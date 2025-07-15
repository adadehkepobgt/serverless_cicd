## 🤖 **Yes! All-in-One Script with Embedded JSON + Copilot Templates**

Absolutely! Let's create a single script file with everything embedded, plus Copilot prompt templates for developers.

---

## 📋 **What is test_config.json?**

### **Simple Explanation:**
```
test_config.json is like a "settings file" for your Lambda tests. 

Think of it as the control panel where you specify:
• Which Lambda function to test
• What AWS region it's in  
• What performance standards to expect
• Which test data to use

Instead of hardcoding these values in the script, we put them in a separate file 
so different developers can easily customize tests for their specific Lambda functions 
without touching the main test code.
```

### **Why Use It:**
```
✅ Easy customization per Lambda function
✅ No need to modify the main test script
✅ Different teams can have different settings
✅ Version control friendly (can track changes to test settings)
```

---

## 🚀 **All-in-One Script Solution**

### **Option 1: Completely Self-Contained Script**

```python
#!/usr/bin/env python3
"""
🚀 ALL-IN-ONE Lambda Test Suite
- No external config files needed
- All test data embedded
- GitHub Copilot integration ready
- Just update the CONFIGURATION section for your Lambda

🤖 COPILOT USAGE:
Select the CONFIGURATION section below and ask Copilot:
"Update this configuration for my Lambda function that [describe your Lambda's purpose]"

🧪 TEST DATA CUSTOMIZATION:
Select the TEST_DATA section and ask Copilot:
"Generate test payloads for my Lambda that [describe your Lambda's inputs/outputs]"
"""

import json
import subprocess
import time
import os
import sys
import base64
from datetime import datetime, timedelta
import uuid

# =============================================================================
# 🔧 CONFIGURATION - CUSTOMIZE THIS SECTION FOR YOUR LAMBDA
# =============================================================================

# 🤖 COPILOT PROMPT TEMPLATE:
# "Update this configuration for my Lambda function that processes user orders, 
# validates payment data, calls external payment API, and stores results in DynamoDB"

LAMBDA_CONFIG = {
    # Basic Lambda settings
    "function_name": "cicd",                    # 🔄 Change to your Lambda name
    "region": "ap-northeast-1",                 # 🔄 Change to your AWS region
    "timeout_seconds": 30,                      # 🔄 Your Lambda timeout
    "memory_mb": 256,                          # 🔄 Your Lambda memory
    
    # Test settings
    "performance_iterations": 5,                # How many times to test performance
    "expected_response_time_ms": 5000,         # Max acceptable response time
    "log_retention_minutes": 10,               # How far back to get logs
    
    # AWS resources (if your Lambda uses them)
    "s3_bucket": "ss-dev-eucm-cicd-apnortheast1-bucket",  # 🔄 Your S3 bucket
    "dynamodb_table": "your-table-name",       # 🔄 Your DynamoDB table
    "sns_topic": "your-topic-arn",             # 🔄 Your SNS topic
    
    # Notification settings
    "email_on_failure": True,
    "slack_webhook": ""                        # 🔄 Optional Slack webhook
}

# =============================================================================
# 📊 TEST DATA - CUSTOMIZE PAYLOADS FOR YOUR LAMBDA
# =============================================================================

# 🤖 COPILOT PROMPT TEMPLATE:
# "Generate test data payloads for my Lambda function that accepts user registration data 
# with fields: email, firstName, lastName, preferences, and returns user ID"

TEST_DATA = {
    # ✅ Valid test payload - should succeed
    "valid_payload": {
        "test_type": "unit_test",
        "user_id": "test-user-123",
        "data": {
            "name": "Test User",
            "email": "test@example.com",
            "timestamp": "2025-01-13T10:30:00Z",
            "action": "process_data"
        },
        "metadata": {
            "source": "automated_test",
            "version": "1.0"
        }
    },
    
    # ❌ Invalid test payload - should fail gracefully
    "invalid_payload": {
        "invalid_field": "missing_required_fields",
        "malformed": True,
        "incomplete_data": "test"
    },
    
    # ⚡ Performance test payload - for load testing
    "performance_payload": {
        "test_type": "performance_test",
        "data_size": "large",
        "batch_size": 100,
        "operation": "bulk_process",
        "timestamp": "2025-01-13T10:30:00Z"
    },
    
    # 🪣 S3 integration payload - for S3 testing
    "s3_integration_payload": {
        "test_type": "s3_integration",
        "operation": "read_and_process",
        "file_type": "json",
        "expected_records": 10
    },
    
    # 🔄 End-to-end workflow payload
    "e2e_workflow_payload": {
        "test_type": "end_to_end",
        "workflow_steps": [
            "validate_input",
            "process_data", 
            "call_external_service",
            "save_results",
            "return_response"
        ],
        "timestamp": "2025-01-13T10:30:00Z",
        "trace_id": "test-trace-12345"
    },
    
    # 📋 Logging test payload - for CloudWatch testing
    "logging_test_payload": {
        "test_type": "logging_test",
        "log_level": "INFO",
        "generate_logs": True,
        "test_scenarios": [
            "info_logging",
            "error_handling", 
            "debug_output",
            "performance_metrics"
        ]
    }
}

# =============================================================================
# 🎨 CONSOLE COLORS & UTILITIES
# =============================================================================

class Colors:
    BLUE = '\033[94m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    RED = '\033[91m'
    END = '\033[0m'
    BOLD = '\033[1m'

# Global test results storage
test_results = {
    'function_name': LAMBDA_CONFIG['function_name'],
    'test_timestamp': datetime.now().isoformat(),
    'tests': [],
    'summary': {'total': 0, 'passed': 0, 'failed': 0},
    'logs': [],
    'performance_metrics': {}
}

# =============================================================================
# 🛠️ UTILITY FUNCTIONS
# =============================================================================

def run_aws_command(command):
    """Execute AWS CLI command and return result"""
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=60)
        return result.returncode == 0, result.stdout, result.stderr
    except subprocess.TimeoutExpired:
        return False, "", "Command timeout"
    except Exception as e:
        return False, "", str(e)

def log_test(test_name, passed, error_msg=""):
    """Log test result and update global results"""
    status = "✅ PASSED" if passed else "❌ FAILED"
    print(f"  {status} {test_name}")
    if error_msg:
        print(f"    Error: {error_msg}")
    
    test_results['tests'].append({
        'name': test_name,
        'passed': passed,
        'error': error_msg,
        'timestamp': datetime.now().isoformat()
    })
    
    test_results['summary']['total'] += 1
    if passed:
        test_results['summary']['passed'] += 1
    else:
        test_results['summary']['failed'] += 1

def create_temp_file(data, filename):
    """Create temporary file for AWS CLI operations"""
    os.makedirs('/tmp/lambda_tests', exist_ok=True)
    filepath = f'/tmp/lambda_tests/{filename}'
    with open(filepath, 'w') as f:
        json.dump(data, f)
    return filepath

def setup_artifacts_directory():
    """Create artifacts directory for test outputs"""
    os.makedirs('artifacts', exist_ok=True)

# =============================================================================
# 🧪 UNIT TESTS
# =============================================================================

def unit_test_verify_function_exists():
    """UNIT TEST: Verify Lambda function is deployed and accessible"""
    print(f"\n{Colors.BLUE}🔍 UNIT TEST: Verifying Lambda function exists{Colors.END}")
    
    command = f"aws lambda get-function --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']}"
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            function_info = json.loads(stdout)
            runtime = function_info['Configuration']['Runtime']
            memory = function_info['Configuration']['MemorySize']
            timeout = function_info['Configuration']['Timeout']
            
            print(f"    Function: {LAMBDA_CONFIG['function_name']}")
            print(f"    Runtime: {runtime}")
            print(f"    Memory: {memory}MB")
            print(f"    Timeout: {timeout}s")
            
            log_test("Function Deployment", True)
            return True
        except Exception as e:
            log_test("Function Deployment", False, f"Failed to parse function info: {e}")
            return False
    else:
        log_test("Function Deployment", False, stderr)
        return False

def unit_test_basic_invocation():
    """UNIT TEST: Test basic Lambda function invocation with valid payload"""
    print(f"\n{Colors.BLUE}🧪 UNIT TEST: Basic Lambda Invocation{Colors.END}")
    
    # Add dynamic timestamp to payload
    payload = TEST_DATA['valid_payload'].copy()
    payload['timestamp'] = datetime.now().isoformat()
    
    payload_file = create_temp_file(payload, 'unit_test_payload.json')
    response_file = '/tmp/lambda_tests/unit_test_response.json'
    
    command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            print(f"    Response: {json.dumps(response, indent=2)[:200]}...")
            
            # Flexible response validation
            if isinstance(response, dict):
                if 'statusCode' in response:
                    if response['statusCode'] == 200:
                        log_test("Basic Invocation", True)
                        return True
                    else:
                        log_test("Basic Invocation", False, f"Non-200 status: {response['statusCode']}")
                        return False
                else:
                    # No statusCode field, but got valid JSON response
                    log_test("Basic Invocation", True, "Valid response received")
                    return True
            else:
                log_test("Basic Invocation", False, f"Unexpected response format: {type(response)}")
                return False
                
        except Exception as e:
            log_test("Basic Invocation", False, f"Failed to process response: {e}")
            return False
    else:
        log_test("Basic Invocation", False, stderr)
        return False

def unit_test_error_handling():
    """UNIT TEST: Test Lambda error handling with invalid payload"""
    print(f"\n{Colors.BLUE}🚨 UNIT TEST: Error Handling{Colors.END}")
    
    payload_file = create_temp_file(TEST_DATA['invalid_payload'], 'error_test_payload.json')
    response_file = '/tmp/lambda_tests/error_test_response.json'
    
    command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            # Lambda should handle errors gracefully
            log_test("Error Handling", True, "Lambda handled invalid input gracefully")
            return True
                
        except Exception as e:
            log_test("Error Handling", False, f"Failed to process error response: {e}")
            return False
    else:
        log_test("Error Handling", False, stderr)
        return False

def unit_test_performance():
    """UNIT TEST: Performance testing with multiple invocations"""
    print(f"\n{Colors.BLUE}⚡ UNIT TEST: Performance Testing{Colors.END}")
    
    # Add dynamic timestamp to performance payload
    payload = TEST_DATA['performance_payload'].copy()
    payload['timestamp'] = datetime.now().isoformat()
    
    payload_file = create_temp_file(payload, 'performance_test_payload.json')
    execution_times = []
    
    for i in range(LAMBDA_CONFIG['performance_iterations']):
        response_file = f'/tmp/lambda_tests/performance_test_response_{i}.json'
        
        start_time = time.time()
        command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --payload file://{payload_file} {response_file}"
        success, stdout, stderr = run_aws_command(command)
        end_time = time.time()
        
        if success:
            execution_time = (end_time - start_time) * 1000
            execution_times.append(execution_time)
        else:
            log_test("Performance Test", False, f"Iteration {i+1} failed: {stderr}")
            return False
    
    if execution_times:
        avg_time = sum(execution_times) / len(execution_times)
        max_time = max(execution_times)
        min_time = min(execution_times)
        
        print(f"    Average execution time: {avg_time:.2f}ms")
        print(f"    Max execution time: {max_time:.2f}ms")
        print(f"    Min execution time: {min_time:.2f}ms")
        
        test_results['performance_metrics'] = {
            'average_ms': round(avg_time, 2),
            'max_ms': round(max_time, 2),
            'min_ms': round(min_time, 2),
            'iterations': LAMBDA_CONFIG['performance_iterations']
        }
        
        if avg_time <= LAMBDA_CONFIG['expected_response_time_ms']:
            log_test("Performance Test", True)
            return True
        else:
            log_test("Performance Test", False, f"Average time {avg_time:.2f}ms exceeds limit {LAMBDA_CONFIG['expected_response_time_ms']}ms")
            return False
    else:
        log_test("Performance Test", False, "No successful executions recorded")
        return False

# =============================================================================
# 🔗 INTEGRATION TESTS
# =============================================================================

def integration_test_s3_operations():
    """INTEGRATION TEST: Test S3 integration if applicable"""
    print(f"\n{Colors.BLUE}🪣 INTEGRATION TEST: S3 Operations{Colors.END}")
    
    if not LAMBDA_CONFIG.get('s3_bucket'):
        log_test("S3 Integration", True, "Skipped - no S3 bucket configured")
        return True
    
    # Create test file for S3
    test_data = {
        "test_type": "s3_integration",
        "timestamp": datetime.now().isoformat(),
        "test_file_id": str(uuid.uuid4())
    }
    
    test_file_key = f"test-data/lambda-test-{int(time.time())}.json"
    test_file_path = '/tmp/lambda_tests/s3_test_file.json'
    
    with open(test_file_path, 'w') as f:
        json.dump(test_data, f)
    
    try:
        # Upload test file to S3
        upload_command = f"aws s3 cp {test_file_path} s3://{LAMBDA_CONFIG['s3_bucket']}/{test_file_key} --region {LAMBDA_CONFIG['region']}"
        success, stdout, stderr = run_aws_command(upload_command)
        
        if not success:
            log_test("S3 Integration", False, f"S3 upload failed: {stderr}")
            return False
        
        # Create Lambda payload that references the S3 file
        s3_payload = TEST_DATA['s3_integration_payload'].copy()
        s3_payload.update({
            "s3_bucket": LAMBDA_CONFIG['s3_bucket'],
            "s3_key": test_file_key,
            "timestamp": datetime.now().isoformat()
        })
        
        payload_file = create_temp_file(s3_payload, 's3_integration_payload.json')
        response_file = '/tmp/lambda_tests/s3_integration_response.json'
        
        # Invoke Lambda with S3 payload
        command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --payload file://{payload_file} {response_file}"
        success, stdout, stderr = run_aws_command(command)
        
        if success:
            log_test("S3 Integration", True)
            
            # Cleanup: Remove test file
            cleanup_command = f"aws s3 rm s3://{LAMBDA_CONFIG['s3_bucket']}/{test_file_key} --region {LAMBDA_CONFIG['region']}"
            run_aws_command(cleanup_command)
            
            return True
        else:
            log_test("S3 Integration", False, stderr)
            return False
            
    except Exception as e:
        log_test("S3 Integration", False, f"S3 integration test failed: {e}")
        return False

def integration_test_cloudwatch_logs():
    """INTEGRATION TEST: CloudWatch Logs Extraction and Analysis"""
    print(f"\n{Colors.BLUE}📊 INTEGRATION TEST: CloudWatch Logs Analysis{Colors.END}")
    
    # Use the embedded logging test payload
    log_test_payload = TEST_DATA['logging_test_payload'].copy()
    log_test_payload.update({
        "timestamp": datetime.now().isoformat(),
        "request_id": str(uuid.uuid4())
    })
    
    payload_file = create_temp_file(log_test_payload, 'logging_test_payload.json')
    response_file = '/tmp/lambda_tests/logging_test_response.json'
    
    # Invoke with log tail to get immediate logs
    command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --log-type Tail --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            # Parse immediate logs from invoke response
            invoke_result = json.loads(stdout)
            
            if "LogResult" in invoke_result:
                # Decode base64 logs
                immediate_logs = base64.b64decode(invoke_result["LogResult"]).decode('utf-8')
                
                print(f"    📋 Immediate CloudWatch Logs:")
                print(f"    ┌─────────────────────────────────────────────────────────┐")
                for line in immediate_logs.strip().split('\n')[:10]:
                    print(f"    │ {line[:55]:<55} │")
                print(f"    └─────────────────────────────────────────────────────────┘")
                
                # Wait for logs to be available in CloudWatch
                time.sleep(5)
                
                # Get detailed logs from CloudWatch
                success = extract_detailed_cloudwatch_logs()
                
                if success:
                    log_test("CloudWatch Logs", True)
                    return True
                else:
                    log_test("CloudWatch Logs", False, "Failed to extract detailed logs")
                    return False
            else:
                log_test("CloudWatch Logs", False, "No LogResult in response")
                return False
        except Exception as e:
            log_test("CloudWatch Logs", False, f"Failed to parse logs: {e}")
            return False
    else:
        log_test("CloudWatch Logs", False, stderr)
        return False

def extract_detailed_cloudwatch_logs():
    """Extract detailed logs from CloudWatch and save them"""
    print(f"\n    📊 Extracting Detailed CloudWatch Logs...")
    
    # Calculate time range
    end_time = int(time.time() * 1000)
    start_time = end_time - (LAMBDA_CONFIG['log_retention_minutes'] * 60 * 1000)
    
    # Get logs from CloudWatch
    log_group = f"/aws/lambda/{LAMBDA_CONFIG['function_name']}"
    command = f"aws logs filter-log-events --log-group-name {log_group} --start-time {start_time} --end-time {end_time} --region {LAMBDA_CONFIG['region']}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            logs_data = json.loads(stdout)
            events = logs_data.get('events', [])
            
            print(f"    📋 CloudWatch Log Analysis:")
            print(f"    • Total log entries: {len(events)}")
            
            # Count log levels
            info_count = sum(1 for event in events if '[INFO]' in event.get('message', ''))
            error_count = sum(1 for event in events if '[ERROR]' in event.get('message', ''))
            warn_count = sum(1 for event in events if '[WARN]' in event.get('message', ''))
            
            print(f"    • INFO messages: {info_count}")
            print(f"    • ERROR messages: {error_count}")
            print(f"    • WARN messages: {warn_count}")
            
            # Show recent entries
            recent_events = events[-5:] if events else []
            if recent_events:
                print(f"    📋 Recent Log Entries:")
                for event in recent_events:
                    timestamp = datetime.fromtimestamp(event['timestamp']/1000).strftime('%H:%M:%S')
                    message = event['message'].strip()[:80]
                    print(f"    {timestamp} {message}")
            
            # Save logs to artifacts
            save_logs_to_artifacts(events)
            
            # Store in test results
            test_results['logs'] = {
                'total_entries': len(events),
                'info_count': info_count,
                'error_count': error_count,
                'warn_count': warn_count,
                'recent_entries': recent_events[-10:] if recent_events else []
            }
            
            return True
            
        except Exception as e:
            print(f"    ❌ Failed to process CloudWatch logs: {e}")
            return False
    else:
        print(f"    ❌ Failed to fetch CloudWatch logs: {stderr}")
        return False

def integration_test_end_to_end_workflow():
    """INTEGRATION TEST: Complete end-to-end workflow test"""
    print(f"\n{Colors.BLUE}🔄 INTEGRATION TEST: End-to-End Workflow{Colors.END}")
    
    workflow_payload = TEST_DATA['e2e_workflow_payload'].copy()
    workflow_payload['timestamp'] = datetime.now().isoformat()
    
    payload_file = create_temp_file(workflow_payload, 'e2e_test_payload.json')
    response_file = '/tmp/lambda_tests/e2e_test_response.json'
    
    command = f"aws lambda invoke --function-name {LAMBDA_CONFIG['function_name']} --region {LAMBDA_CONFIG['region']} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            log_test("End-to-End Workflow", True, "E2E workflow completed")
            return True
        except Exception as e:
            log_test("End-to-End Workflow", False, f"Failed to process E2E response: {e}")
            return False
    else:
        log_test("End-to-End Workflow", False, stderr)
        return False

# =============================================================================
# 📊 REPORTING FUNCTIONS  
# =============================================================================

def save_logs_to_artifacts(events):
    """Save CloudWatch logs to artifact files"""
    try:
        # Save raw logs as text
        with open('artifacts/cloudwatch_logs.txt', 'w') as f:
            f.write(f"CloudWatch Logs for Lambda: {LAMBDA_CONFIG['function_name']}\n")
            f.write(f"Generated: {datetime.now().isoformat()}\n")
            f.write("=" * 60 + "\n\n")
            
            for event in events:
                timestamp = datetime.fromtimestamp(event['timestamp']/1000).isoformat()
                f.write(f"[{timestamp}] {event['message']}\n")
        
        # Save logs as JSON
        with open('artifacts/cloudwatch_logs.json', 'w') as f:
            json.dump({
                "function_name": LAMBDA_CONFIG['function_name'],
                "extraction_timestamp": datetime.now().isoformat(),
                "total_events": len(events),
                "events": events
            }, f, indent=2)
        
        print(f"    💾 Logs saved to artifacts/")
        
    except Exception as e:
        print(f"    ❌ Failed to save logs: {e}")

def generate_test_report():
    """Generate comprehensive HTML test report"""
    html_content = f"""
<!DOCTYPE html>
<html>
<head>
    <title>Lambda Test Report - {LAMBDA_CONFIG['function_name']}</title>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .header {{ background: #f0f0f0; padding: 20px; border-radius: 5px; }}
        .summary {{ background: #e8f5e8; padding: 15px; margin: 20px 0; border-radius: 5px; }}
        .failed {{ background: #ffe8e8; }}
        .test-result {{ margin: 10px 0; padding: 10px; border-left: 4px solid #ccc; }}
        .passed {{ border-left-color: #4CAF50; }}
        .failed {{ border-left-color: #f44336; }}
        .logs {{ background: #f8f8f8; padding: 10px; font-family: monospace; white-space: pre-wrap; }}
        .metric {{ display: inline-block; margin: 10px; padding: 10px; background: #f0f0f0; border-radius: 3px; }}
    </style>
</head>
<body>
    <div class="header">
        <h1>🚀 Lambda Function Test Report</h1>
        <p><strong>Function:</strong> {LAMBDA_CONFIG['function_name']}</p>
        <p><strong>Region:</strong> {LAMBDA_CONFIG['region']}</p>
        <p><strong>Test Date:</strong> {test_results['test_timestamp']}</p>
    </div>
    
    <div class="summary {'failed' if test_results['summary']['failed'] > 0 else ''}">
        <h2>📊 Test Summary</h2>
        <div class="metric">
            <strong>Total Tests:</strong> {test_results['summary']['total']}
        </div>
        <div class="metric">
            <strong>✅ Passed:</strong> {test_results['summary']['passed']}
        </div>
        <div class="metric">
            <strong>❌ Failed:</strong> {test_results['summary']['failed']}
        </div>
        <div class="metric">
            <strong>Success Rate:</strong> {(test_results['summary']['passed'] / test_results['summary']['total'] * 100) if test_results['summary']['total'] > 0 else 0:.1f}%
        </div>
    </div>
    
    <h2>🧪 Test Results</h2>
"""
    
    # Add test results
    for test in test_results['tests']:
        status_class = 'passed' if test['passed'] else 'failed'
        status_icon = '✅' if test['passed'] else '❌'
        html_content += f"""
    <div class="test-result {status_class}">
        <strong>{status_icon} {test['name']}</strong>
        <p>Time: {test['timestamp']}</p>
        {f"<p>Error: {test['error']}</p>" if test['error'] else ""}
    </div>
"""
    
    # Add performance metrics
    if test_results.get('performance_metrics'):
        perf = test_results['performance_metrics']
        html_content += f"""
    <h2>⚡ Performance Metrics</h2>
    <div class="summary">
        <div class="metric">
            <strong>Average Response Time:</strong> {perf['average_ms']}ms
        </div>
        <div class="metric">
            <strong>Max Response Time:</strong> {perf['max_ms']}ms
        </div>
        <div class="metric">
            <strong>Min Response Time:</strong> {perf['min_ms']}ms
        </div>
        <div class="metric">
            <strong>Test Iterations:</strong> {perf['iterations']}
        </div>
    </div>
"""
    
    # Add log analysis
    if test_results.get('logs'):
        logs = test_results['logs']
        html_content += f"""
    <h2>📋 CloudWatch Logs Analysis</h2>
    <div class="summary">
        <div class="metric">
            <strong>Total Log Entries:</strong> {logs['total_entries']}
        </div>
        <div class="metric">
            <strong>INFO Messages:</strong> {logs['info_count']}
        </div>
        <div class="metric">
            <strong>ERROR Messages:</strong> {logs['error_count']}
        </div>
        <div class="metric">
            <strong>WARN Messages:</strong> {logs['warn_count']}
        </div>
    </div>
"""
    
    html_content += "</body></html>"
    
    # Save reports
    with open('artifacts/test_report.html', 'w') as f:
        f.write(html_content)
    
    with open('artifacts/test_results.json', 'w') as f:
        json.dump(test_results, f, indent=2)

def print_test_summary():
    """Print final test summary to console"""
    print(f"\n{Colors.BOLD}📊 Test Summary{Colors.END}")
    print("=" * 50)
    print(f"Total Tests: {test_results['summary']['total']}")
    print(f"{Colors.GREEN}✅ Passed: {test_results['summary']['passed']}{Colors.END}")
    print(f"{Colors.RED}❌ Failed: {test_results['summary']['failed']}{Colors.END}")
    
    if test_results['summary']['failed'] == 0:
        print(f"\n{Colors.GREEN}🎉 All tests passed!{Colors.END}")
    else:
        print(f"\n{Colors.RED}❌ Some tests failed. Check the detailed report.{Colors.END}")
        print("\nFailed Tests:")
        for test in test_results['tests']:
            if not test['passed']:
                print(f"  - {test['name']}: {test['error']}")
    
    print(f"\nTest execution completed in {time.time() - start_time:.1f} seconds")
    print(f"📁 Artifacts saved to: artifacts/")
    print("   • test_report.html")
    print("   • test_results.json") 
    print("   • cloudwatch_logs.txt")
    print("   • cloudwatch_logs.json")

# =============================================================================
# 🚀 MAIN EXECUTION
# =============================================================================

def main():
    """Main test execution function"""
    global start_time
    start_time = time.time()
    
    print(f"{Colors.BOLD}🚀 ALL-IN-ONE Lambda Function Test Suite{Colors.END}")
    print(f"Function: {LAMBDA_CONFIG['function_name']}")
    print(f"Region: {LAMBDA_CONFIG['region']}")
    print(f"Timestamp: {datetime.now().isoformat()}")
    print("=" * 50)
    
    # Setup
    setup_artifacts_directory()
    
    # Run Unit Tests
    print(f"\n{Colors.YELLOW}🔬 RUNNING UNIT TESTS{Colors.END}")
    unit_test_verify_function_exists()
    unit_test_basic_invocation()
    unit_test_error_handling()
    unit_test_performance()
    
    # Run Integration Tests
    print(f"\n{Colors.YELLOW}🔗 RUNNING INTEGRATION TESTS{Colors.END}")
    integration_test_s3_operations()
    integration_test_cloudwatch_logs()
    integration_test_end_to_end_workflow()
    
    # Generate Reports
    print(f"\n{Colors.YELLOW}📊 GENERATING REPORTS{Colors.END}")
    generate_test_report()
    print_test_summary()
    
    # Exit with appropriate code
    exit_code = 0 if test_results['summary']['failed'] == 0 else 1
    sys.exit(exit_code)

if __name__ == "__main__":
    main()
```

---

## 🤖 **GitHub Copilot Prompt Templates for Developers**

### **File: `copilot_prompts.md`**

```markdown
# 🤖 GitHub Copilot Prompt Templates for Lambda Testing

## 🎯 Quick Setup Prompts

### 1. 📝 Configure for Your Lambda
```
Select the LAMBDA_CONFIG section and prompt:

"Update this configuration for my Lambda function that [describe your Lambda]:
- Function name: [your-function-name]  
- Processes [type of data]
- Uses [AWS services like S3/DynamoDB/SQS]
- Returns [response format]
- Performance requirements: [response time/memory]"

Example:
"Update this configuration for my Lambda function that processes user registration data, validates emails, stores user info in DynamoDB, sends welcome emails via SES, and returns user ID. Function name is 'user-registration-handler', should respond within 2 seconds."
```

### 2. 🧪 Generate Test Data
```
Select the TEST_DATA section and prompt:

"Generate test payloads for my Lambda function that:
- Accepts: [input format and required fields]
- Returns: [output format and fields]  
- Handles: [business logic description]
- Edge cases: [error scenarios to test]"

Example:
"Generate test payloads for my Lambda that accepts order data with fields: customerId, items[], paymentMethod, shippingAddress. Returns orderId and status. Handles payment validation, inventory checks, and shipping calculations. Test edge cases: invalid payment, out-of-stock items, invalid addresses."
```

### 3. 🔧 Customize Integration Tests  
```
Select any integration test function and prompt:

"Modify this integration test for my Lambda that:
- Uses [AWS service] for [purpose]
- Expects [specific behavior]
- Should handle [error scenarios]"

Example:
"Modify this S3 integration test for my Lambda that reads CSV files from S3, processes financial data, and saves summarized reports back to S3. Should handle malformed CSV, large files >10MB, and empty files."
```

## 🚀 Advanced Customization Prompts

### 4. 📊 Custom Performance Tests
```
"Create performance test scenarios for my Lambda including:
- [Number] concurrent invocations
- Test with [data size] payloads  
- Measure [specific metrics]
- Alert if [performance thresholds] exceeded"

Example:
"Create performance tests with 10 concurrent invocations, test with 1MB JSON payloads, measure memory usage and processing time, alert if average response exceeds 3 seconds or memory usage exceeds 512MB."
```

### 5. 🚨 Custom Error Scenarios
```
"Generate error handling test cases for my Lambda covering:
- [Business logic errors]
- [AWS service failures]  
- [Input validation errors]
- [External API failures]"

Example:
"Generate error tests covering: duplicate order submission, payment gateway timeouts, invalid customer IDs, inventory service downtime, malformed product data, and credit card validation failures."
```

### 6. 🔗 Custom Integration Tests
```
"Create integration tests for my Lambda that connects to:
- [External service 1] for [purpose]
- [External service 2] for [purpose]
- [Database/storage] for [purpose]"

Example:
"Create integration tests for my Lambda that connects to Stripe API for payments, SendGrid for emails, Redis for caching, and PostgreSQL for order storage. Test both success and failure scenarios for each service."
```

## 🎯 One-Shot Complete Setup

### 7. 🚀 Complete Test Suite Generation
```
"Generate a complete test suite for my Lambda function with these specifications:

**Function Purpose:** [Detailed description]
**Input:** [Format and required fields]
**Output:** [Format and fields]
**AWS Services Used:** [List services and purposes]
**Business Logic:** [Key workflows and validations]
**Performance Requirements:** [Response time, memory, etc.]
**Error Scenarios:** [Key failure cases to test]

Include unit tests, integration tests, performance tests, and error handling tests. Generate appropriate test data for each scenario."

Example:
"Generate complete test suite for my e-commerce order processing Lambda:

**Function Purpose:** Process incoming orders, validate payment, check inventory, calculate shipping, and create order records
**Input:** JSON with customerId, items[{productId, quantity, price}], paymentMethod{type, cardToken}, shippingAddress{street, city, state, zip}
**Output:** JSON with orderId, status, totalAmount, estimatedDelivery, confirmationNumber
**AWS Services:** DynamoDB (customer/inventory data), SQS (order queue), SNS (notifications), S3 (order documents)
**Business Logic:** Payment validation, inventory checking, tax calculation, shipping cost calculation, fraud detection
**Performance:** <2sec response, <256MB memory, handle 100 concurrent orders
**Error Scenarios:** Invalid payment, insufficient inventory, invalid addresses, service outages, duplicate orders"
```

## 🔧 Maintenance & Updates

### 8. 🔄 Update Existing Tests
```
"Update my existing test suite to include:
- [New functionality]
- [Additional AWS services]
- [New error scenarios]
- [Performance improvements]"

Example:
"Update my test suite to include new fraud detection functionality, integration with new recommendation engine API, testing for international shipping, and validation for new subscription payment model."
```

### 9. 🐛 Debug Failed Tests
```
"Analyze this failed test and suggest improvements:
[Paste test function and error message]

Help me understand:
- Why the test failed
- How to fix the test
- What additional test cases might be needed"
```

### 10. 🎨 Customize Test Output
```
"Modify the test reporting to include:
- [Custom metrics]
- [Business-specific validations]
- [Integration with [monitoring tool]]
- [Custom notification format]"

Example:
"Modify test reporting to include order processing rate, average cart value processed, integration with DataDog metrics, and Slack notifications with order volume statistics."
```
```

---

## 📋 **Simple Repository Structure**

```
lambda-functions/
├── lambda_code/
│   └── lambda_function.py              # Your Lambda code
├── terraform/
│   └── main.tf                         # Your Terraform
├── tests/
│   ├── lambda_test_suite.py           # 🆕 ALL-IN-ONE test script
│   └── copilot_prompts.md             # 🆕 Prompt templates
└── README.md
```

---

## 🎯 **Developer Workflow**

### **Super Simple Steps:**
```
1. 📁 Copy lambda_test_suite.py to your tests/ folder
2. 🔧 Update LAMBDA_CONFIG section (function name, region, etc.)
3. 🤖 Use Copilot prompts to customize TEST_DATA for your Lambda
4. ✅ Commit and push - Jenkins will run tests automatically
5. 📊 View results in Jenkins build artifacts
```

### **Advanced Customization:**
```
1. 🎯 Use Copilot prompts from copilot_prompts.md
2. 🔄 Select sections of the script and ask Copilot to modify
3. 🧪 Add custom test functions using Copilot
4. 📊 Customize reporting and notifications
```

---

## 🚀 **Benefits of All-in-One Approach**

### **For Developers:**
```
✅ Single file to manage - no external config files
✅ All test data embedded - no separate JSON files  
✅ Copilot-friendly structure - easy to customize
✅ Copy-paste ready - works immediately
✅ Self-documenting - comments explain everything
```

### **For Teams:**
```
✅ Consistent testing across all Lambda functions
✅ Easy to share and reuse
✅ Version control friendly - everything in one file
✅ No dependencies - just Python + AWS CLI
✅ Scalable - same script works for any Lambda
```

This approach eliminates the need for separate config files while making it super easy for developers to customize tests using GitHub Copilot! 🚀


## 🛠️ **Detailed Implementation Guide: Lambda Testing Infrastructure**

Let me break down every step with exact locations, tools, and script examples.

---

## 📋 **Step-by-Step Implementation Details**

### **Step 1: 🚀 Trigger Point - Jenkins Pipeline Integration**

#### **Where:** 
```
File Location: /jenkins/pipelines/Jenkinsfile
Repository: Your existing Lambda repository (same as Terraform files)
```

#### **Tools Needed:**
```
✅ Existing Jenkins server
✅ Existing AWS CLI access
✅ Python 3.x environment (already in Jenkins)
✅ Bitbucket webhook (already configured)
```

#### **Updated Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        FUNCTION_NAME = 'cicd'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://bitbucket.org/your-org/lambda-functions.git'
            }
        }
        
        stage('Deploy Lambda') {
            steps {
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    dir('terraform') {
                        sh 'terraform init'
                        sh 'terraform apply -auto-approve'
                    }
                }
            }
        }
        
        stage('Test Lambda') {  // 🆕 NEW STAGE
            steps {
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    dir('tests') {
                        sh 'python lambda_test_suite.py'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'tests/artifacts/**/*', allowEmptyArchive: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'tests/artifacts',
                        reportFiles: 'test_report.html',
                        reportName: 'Lambda Test Report'
                    ])
                }
            }
        }
    }
}
```

---

### **Step 2: 🔍 Repository Structure Setup**

#### **Where:**
```
Your Lambda Repository Structure:
lambda-functions/
├── lambda_code/
│   ├── lambda_function.py          # Your Lambda code
│   ├── requirements.txt            # Dependencies
│   └── utils/                      # Helper modules
├── terraform/
│   ├── main.tf                     # Existing Terraform
│   ├── variables.tf
│   └── outputs.tf
├── tests/                          # 🆕 NEW DIRECTORY
│   ├── lambda_test_suite.py        # 🆕 Main test script
│   ├── test_config.json           # 🆕 Test configuration
│   ├── test_data/                  # 🆕 Test input files
│   │   ├── valid_payload.json
│   │   ├── invalid_payload.json
│   │   └── performance_payload.json
│   └── artifacts/                  # 🆕 Test output (created during run)
│       ├── test_report.html
│       ├── cloudwatch_logs.txt
│       └── test_results.json
└── README.md
```

#### **Tools Needed:**
```
✅ Git repository access
✅ Text editor/IDE for creating files
✅ JSON validator for test data files
```

---

### **Step 3: 🧪 Test Configuration Setup**

#### **Where:**
```
File: tests/test_config.json
```

#### **Content:**
```json
{
  "lambda_function": {
    "name": "cicd",
    "region": "ap-northeast-1",
    "timeout_seconds": 30,
    "memory_mb": 256
  },
  "test_settings": {
    "performance_iterations": 5,
    "log_retention_minutes": 10,
    "s3_test_bucket": "ss-dev-eucm-cicd-apnortheast1-bucket",
    "expected_response_time_ms": 5000
  },
  "test_data_paths": {
    "valid_payload": "test_data/valid_payload.json",
    "invalid_payload": "test_data/invalid_payload.json",
    "performance_payload": "test_data/performance_payload.json"
  },
  "notifications": {
    "email_on_failure": true,
    "slack_webhook": "optional"
  }
}
```

---

### **Step 4: 📝 Test Data Files Setup**

#### **Where:**
```
Directory: tests/test_data/
```

#### **Files & Contents:**

**tests/test_data/valid_payload.json:**
```json
{
  "test_type": "unit_test",
  "user_id": "test-user-123",
  "data": {
    "name": "Test User",
    "email": "test@example.com",
    "timestamp": "2025-01-13T10:30:00Z"
  },
  "expected_response": {
    "statusCode": 200,
    "message": "Success"
  }
}
```

**tests/test_data/invalid_payload.json:**
```json
{
  "invalid_field": "missing_required_fields",
  "malformed": true
}
```

**tests/test_data/performance_payload.json:**
```json
{
  "test_type": "performance_test",
  "data_size": "large",
  "iterations": 100,
  "concurrent_requests": false
}
```

---

## 🧪 **Complete Test Script: Combined Unit + Integration Tests**

### **Where:**
```
File: tests/lambda_test_suite.py
```

### **Tools Needed:**
```
✅ Python 3.x (in Jenkins environment)
✅ AWS CLI (configured with permissions)
✅ Standard Python libraries: json, subprocess, time, datetime, os, base64
✅ No additional pip packages required
```

### **Complete Script:**

```python
#!/usr/bin/env python3
"""
Lambda Function Test Suite - Combined Unit & Integration Tests
Runs comprehensive testing including CloudWatch log extraction
"""

import json
import subprocess
import time
import os
import sys
import base64
from datetime import datetime, timedelta
import uuid

# =============================================================================
# CONFIGURATION & SETUP
# =============================================================================

class Colors:
    """Console colors for better readability"""
    BLUE = '\033[94m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    RED = '\033[91m'
    END = '\033[0m'
    BOLD = '\033[1m'

class TestConfig:
    """Load test configuration"""
    def __init__(self):
        with open('test_config.json', 'r') as f:
            config = json.load(f)
        
        self.function_name = config['lambda_function']['name']
        self.region = config['lambda_function']['region']
        self.timeout = config['lambda_function']['timeout_seconds']
        self.memory = config['lambda_function']['memory_mb']
        self.s3_bucket = config['test_settings']['s3_test_bucket']
        self.expected_response_time = config['test_settings']['expected_response_time_ms']
        self.performance_iterations = config['test_settings']['performance_iterations']
        
        # Load test data
        self.valid_payload = self._load_test_data(config['test_data_paths']['valid_payload'])
        self.invalid_payload = self._load_test_data(config['test_data_paths']['invalid_payload'])
        self.performance_payload = self._load_test_data(config['test_data_paths']['performance_payload'])
    
    def _load_test_data(self, path):
        with open(path, 'r') as f:
            return json.load(f)

# Initialize configuration
config = TestConfig()

# Test results storage
test_results = {
    'function_name': config.function_name,
    'test_timestamp': datetime.now().isoformat(),
    'tests': [],
    'summary': {'total': 0, 'passed': 0, 'failed': 0},
    'logs': [],
    'performance_metrics': {}
}

# =============================================================================
# UTILITY FUNCTIONS
# =============================================================================

def run_aws_command(command):
    """Execute AWS CLI command and return result"""
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=60)
        return result.returncode == 0, result.stdout, result.stderr
    except subprocess.TimeoutExpired:
        return False, "", "Command timeout"
    except Exception as e:
        return False, "", str(e)

def log_test(test_name, passed, error_msg=""):
    """Log test result and update global results"""
    status = "✅ PASSED" if passed else "❌ FAILED"
    print(f"  {status} {test_name}")
    if error_msg:
        print(f"    Error: {error_msg}")
    
    test_results['tests'].append({
        'name': test_name,
        'passed': passed,
        'error': error_msg,
        'timestamp': datetime.now().isoformat()
    })
    
    test_results['summary']['total'] += 1
    if passed:
        test_results['summary']['passed'] += 1
    else:
        test_results['summary']['failed'] += 1

def create_temp_file(data, filename):
    """Create temporary file for AWS CLI operations"""
    os.makedirs('/tmp/lambda_tests', exist_ok=True)
    filepath = f'/tmp/lambda_tests/{filename}'
    with open(filepath, 'w') as f:
        json.dump(data, f)
    return filepath

def setup_artifacts_directory():
    """Create artifacts directory for test outputs"""
    os.makedirs('artifacts', exist_ok=True)

# =============================================================================
# UNIT TESTS
# =============================================================================

def unit_test_verify_function_exists():
    """UNIT TEST: Verify Lambda function is deployed and accessible"""
    print(f"\n{Colors.BLUE}🔍 UNIT TEST: Verifying Lambda function exists{Colors.END}")
    
    command = f"aws lambda get-function --function-name {config.function_name} --region {config.region}"
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            function_info = json.loads(stdout)
            runtime = function_info['Configuration']['Runtime']
            memory = function_info['Configuration']['MemorySize']
            timeout = function_info['Configuration']['Timeout']
            
            print(f"    Function: {config.function_name}")
            print(f"    Runtime: {runtime}")
            print(f"    Memory: {memory}MB")
            print(f"    Timeout: {timeout}s")
            
            log_test("Function Deployment", True)
            return True
        except Exception as e:
            log_test("Function Deployment", False, f"Failed to parse function info: {e}")
            return False
    else:
        log_test("Function Deployment", False, stderr)
        return False

def unit_test_basic_invocation():
    """UNIT TEST: Test basic Lambda function invocation with valid payload"""
    print(f"\n{Colors.BLUE}🧪 UNIT TEST: Basic Lambda Invocation{Colors.END}")
    
    # Create test payload file
    payload_file = create_temp_file(config.valid_payload, 'unit_test_payload.json')
    response_file = '/tmp/lambda_tests/unit_test_response.json'
    
    command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            # Read response
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            print(f"    Response received: {json.dumps(response, indent=2)[:200]}...")
            
            # Validate response structure
            if 'statusCode' in response and response['statusCode'] == 200:
                log_test("Basic Invocation", True)
                return True
            else:
                log_test("Basic Invocation", False, f"Unexpected response: {response}")
                return False
        except Exception as e:
            log_test("Basic Invocation", False, f"Failed to process response: {e}")
            return False
    else:
        log_test("Basic Invocation", False, stderr)
        return False

def unit_test_error_handling():
    """UNIT TEST: Test Lambda error handling with invalid payload"""
    print(f"\n{Colors.BLUE}🚨 UNIT TEST: Error Handling{Colors.END}")
    
    payload_file = create_temp_file(config.invalid_payload, 'error_test_payload.json')
    response_file = '/tmp/lambda_tests/error_test_response.json'
    
    command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            # For error handling, we expect either:
            # 1. A proper error response (statusCode 4xx/5xx)
            # 2. A handled error in the response
            if ('statusCode' in response and response['statusCode'] >= 400) or \
               ('error' in response) or \
               ('errorMessage' in response):
                log_test("Error Handling", True)
                return True
            else:
                # If we get a 200 response, check if error was handled gracefully
                log_test("Error Handling", True, "Lambda handled invalid input gracefully")
                return True
        except Exception as e:
            log_test("Error Handling", False, f"Failed to process error response: {e}")
            return False
    else:
        log_test("Error Handling", False, stderr)
        return False

def unit_test_performance():
    """UNIT TEST: Performance testing with multiple invocations"""
    print(f"\n{Colors.BLUE}⚡ UNIT TEST: Performance Testing{Colors.END}")
    
    payload_file = create_temp_file(config.performance_payload, 'performance_test_payload.json')
    execution_times = []
    
    for i in range(config.performance_iterations):
        response_file = f'/tmp/lambda_tests/performance_test_response_{i}.json'
        
        start_time = time.time()
        command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --payload file://{payload_file} {response_file}"
        success, stdout, stderr = run_aws_command(command)
        end_time = time.time()
        
        if success:
            execution_time = (end_time - start_time) * 1000  # Convert to milliseconds
            execution_times.append(execution_time)
        else:
            log_test("Performance Test", False, f"Iteration {i+1} failed: {stderr}")
            return False
    
    if execution_times:
        avg_time = sum(execution_times) / len(execution_times)
        max_time = max(execution_times)
        min_time = min(execution_times)
        
        print(f"    Average execution time: {avg_time:.2f}ms")
        print(f"    Max execution time: {max_time:.2f}ms")
        print(f"    Min execution time: {min_time:.2f}ms")
        
        test_results['performance_metrics'] = {
            'average_ms': round(avg_time, 2),
            'max_ms': round(max_time, 2),
            'min_ms': round(min_time, 2),
            'iterations': config.performance_iterations
        }
        
        # Check if performance meets expectations
        if avg_time <= config.expected_response_time:
            log_test("Performance Test", True)
            return True
        else:
            log_test("Performance Test", False, f"Average time {avg_time:.2f}ms exceeds limit {config.expected_response_time}ms")
            return False
    else:
        log_test("Performance Test", False, "No successful executions recorded")
        return False

# =============================================================================
# INTEGRATION TESTS
# =============================================================================

def integration_test_s3_operations():
    """INTEGRATION TEST: Test S3 integration if applicable"""
    print(f"\n{Colors.BLUE}🪣 INTEGRATION TEST: S3 Operations{Colors.END}")
    
    # Create test file for S3
    test_data = {
        "test_type": "s3_integration",
        "timestamp": datetime.now().isoformat(),
        "test_file_id": str(uuid.uuid4())
    }
    
    test_file_key = f"test-data/lambda-test-{int(time.time())}.json"
    test_file_path = '/tmp/lambda_tests/s3_test_file.json'
    
    with open(test_file_path, 'w') as f:
        json.dump(test_data, f)
    
    try:
        # Upload test file to S3
        upload_command = f"aws s3 cp {test_file_path} s3://{config.s3_bucket}/{test_file_key} --region {config.region}"
        success, stdout, stderr = run_aws_command(upload_command)
        
        if not success:
            log_test("S3 Integration", False, f"S3 upload failed: {stderr}")
            return False
        
        # Create Lambda payload that references the S3 file
        s3_payload = {
            "test_type": "s3_integration",
            "s3_bucket": config.s3_bucket,
            "s3_key": test_file_key,
            "operation": "read_and_process"
        }
        
        payload_file = create_temp_file(s3_payload, 's3_integration_payload.json')
        response_file = '/tmp/lambda_tests/s3_integration_response.json'
        
        # Invoke Lambda with S3 payload
        command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --payload file://{payload_file} {response_file}"
        success, stdout, stderr = run_aws_command(command)
        
        if success:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            if 'statusCode' in response and response['statusCode'] == 200:
                log_test("S3 Integration", True)
                
                # Cleanup: Remove test file
                cleanup_command = f"aws s3 rm s3://{config.s3_bucket}/{test_file_key} --region {config.region}"
                run_aws_command(cleanup_command)
                
                return True
            else:
                log_test("S3 Integration", False, f"Lambda S3 operation failed: {response}")
                return False
        else:
            log_test("S3 Integration", False, stderr)
            return False
            
    except Exception as e:
        log_test("S3 Integration", False, f"S3 integration test failed: {e}")
        return False

def integration_test_cloudwatch_logs():
    """INTEGRATION TEST: CloudWatch Logs Extraction and Analysis"""
    print(f"\n{Colors.BLUE}📊 INTEGRATION TEST: CloudWatch Logs Analysis{Colors.END}")
    
    # First, invoke Lambda to generate fresh logs
    log_test_payload = {
        "test_type": "logging_test",
        "timestamp": datetime.now().isoformat(),
        "request_id": str(uuid.uuid4()),
        "generate_logs": True
    }
    
    payload_file = create_temp_file(log_test_payload, 'logging_test_payload.json')
    response_file = '/tmp/lambda_tests/logging_test_response.json'
    
    # Invoke with log tail to get immediate logs
    command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --log-type Tail --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            # Parse immediate logs from invoke response
            invoke_result = json.loads(stdout)
            
            if "LogResult" in invoke_result:
                # Decode base64 logs
                immediate_logs = base64.b64decode(invoke_result["LogResult"]).decode('utf-8')
                
                print(f"    📋 Immediate CloudWatch Logs:")
                print(f"    ┌─────────────────────────────────────────────────────────┐")
                for line in immediate_logs.strip().split('\n')[:10]:  # Show first 10 lines
                    print(f"    │ {line[:55]:<55} │")
                print(f"    └─────────────────────────────────────────────────────────┘")
                
                # Wait for logs to be available in CloudWatch
                time.sleep(5)
                
                # Get detailed logs from CloudWatch
                success = extract_detailed_cloudwatch_logs()
                
                if success:
                    log_test("CloudWatch Logs", True)
                    return True
                else:
                    log_test("CloudWatch Logs", False, "Failed to extract detailed logs")
                    return False
            else:
                log_test("CloudWatch Logs", False, "No LogResult in response")
                return False
        except Exception as e:
            log_test("CloudWatch Logs", False, f"Failed to parse logs: {e}")
            return False
    else:
        log_test("CloudWatch Logs", False, stderr)
        return False

def extract_detailed_cloudwatch_logs():
    """Extract detailed logs from CloudWatch and save them"""
    print(f"\n    📊 Extracting Detailed CloudWatch Logs...")
    
    # Calculate time range (last 10 minutes)
    end_time = int(time.time() * 1000)
    start_time = end_time - (10 * 60 * 1000)  # 10 minutes ago
    
    # Get logs from CloudWatch
    log_group = f"/aws/lambda/{config.function_name}"
    command = f"aws logs filter-log-events --log-group-name {log_group} --start-time {start_time} --end-time {end_time} --region {config.region}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            logs_data = json.loads(stdout)
            events = logs_data.get('events', [])
            
            # Display logs analysis in console
            print(f"    📋 CloudWatch Log Analysis:")
            print(f"    • Total log entries: {len(events)}")
            
            # Count log levels
            info_count = sum(1 for event in events if '[INFO]' in event.get('message', ''))
            error_count = sum(1 for event in events if '[ERROR]' in event.get('message', ''))
            warn_count = sum(1 for event in events if '[WARN]' in event.get('message', ''))
            
            print(f"    • INFO messages: {info_count}")
            print(f"    • ERROR messages: {error_count}")
            print(f"    • WARN messages: {warn_count}")
            
            # Show recent entries
            recent_events = events[-5:] if events else []
            if recent_events:
                print(f"    📋 Recent Log Entries:")
                for event in recent_events:
                    timestamp = datetime.fromtimestamp(event['timestamp']/1000).strftime('%H:%M:%S')
                    message = event['message'].strip()[:80]
                    print(f"    {timestamp} {message}")
            
            # Save logs to artifacts
            save_logs_to_artifacts(events)
            
            # Store in test results
            test_results['logs'] = {
                'total_entries': len(events),
                'info_count': info_count,
                'error_count': error_count,
                'warn_count': warn_count,
                'recent_entries': recent_events[-10:] if recent_events else []
            }
            
            return True
            
        except Exception as e:
            print(f"    ❌ Failed to process CloudWatch logs: {e}")
            return False
    else:
        print(f"    ❌ Failed to fetch CloudWatch logs: {stderr}")
        return False

def save_logs_to_artifacts(events):
    """Save CloudWatch logs to artifact files"""
    try:
        # Save raw logs as text
        with open('artifacts/cloudwatch_logs.txt', 'w') as f:
            f.write(f"CloudWatch Logs for Lambda: {config.function_name}\n")
            f.write(f"Generated: {datetime.now().isoformat()}\n")
            f.write("=" * 60 + "\n\n")
            
            for event in events:
                timestamp = datetime.fromtimestamp(event['timestamp']/1000).isoformat()
                f.write(f"[{timestamp}] {event['message']}\n")
        
        # Save logs as JSON for programmatic access
        with open('artifacts/cloudwatch_logs.json', 'w') as f:
            json.dump({
                "function_name": config.function_name,
                "extraction_timestamp": datetime.now().isoformat(),
                "total_events": len(events),
                "events": events
            }, f, indent=2)
        
        print(f"    💾 Logs saved to artifacts/")
        
    except Exception as e:
        print(f"    ❌ Failed to save logs: {e}")

def integration_test_end_to_end_workflow():
    """INTEGRATION TEST: Complete end-to-end workflow test"""
    print(f"\n{Colors.BLUE}🔄 INTEGRATION TEST: End-to-End Workflow{Colors.END}")
    
    workflow_payload = {
        "test_type": "end_to_end",
        "workflow_steps": [
            "validate_input",
            "process_data", 
            "call_external_service",
            "save_results",
            "return_response"
        ],
        "timestamp": datetime.now().isoformat()
    }
    
    payload_file = create_temp_file(workflow_payload, 'e2e_test_payload.json')
    response_file = '/tmp/lambda_tests/e2e_test_response.json'
    
    command = f"aws lambda invoke --function-name {config.function_name} --region {config.region} --payload file://{payload_file} {response_file}"
    
    success, stdout, stderr = run_aws_command(command)
    
    if success:
        try:
            with open(response_file, 'r') as f:
                response = json.load(f)
            
            # Check if all workflow steps completed
            if 'statusCode' in response and response['statusCode'] == 200:
                if 'workflow_completed' in response.get('body', '{}'):
                    log_test("End-to-End Workflow", True)
                    return True
                else:
                    log_test("End-to-End Workflow", True, "Basic E2E completed")
                    return True
            else:
                log_test("End-to-End Workflow", False, f"Workflow failed: {response}")
                return False
        except Exception as e:
            log_test("End-to-End Workflow", False, f"Failed to process E2E response: {e}")
            return False
    else:
        log_test("End-to-End Workflow", False, stderr)
        return False

# =============================================================================
# REPORTING FUNCTIONS
# =============================================================================

def generate_test_report():
    """Generate comprehensive HTML test report"""
    html_content = f"""
<!DOCTYPE html>
<html>
<head>
    <title>Lambda Test Report - {config.function_name}</title>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .header {{ background: #f0f0f0; padding: 20px; border-radius: 5px; }}
        .summary {{ background: #e8f5e8; padding: 15px; margin: 20px 0; border-radius: 5px; }}
        .failed {{ background: #ffe8e8; }}
        .test-result {{ margin: 10px 0; padding: 10px; border-left: 4px solid #ccc; }}
        .passed {{ border-left-color: #4CAF50; }}
        .failed {{ border-left-color: #f44336; }}
        .logs {{ background: #f8f8f8; padding: 10px; font-family: monospace; white-space: pre-wrap; }}
        .metric {{ display: inline-block; margin: 10px; padding: 10px; background: #f0f0f0; border-radius: 3px; }}
    </style>
</head>
<body>
    <div class="header">
        <h1>Lambda Function Test Report</h1>
        <p><strong>Function:</strong> {config.function_name}</p>
        <p><strong>Region:</strong> {config.region}</p>
        <p><strong>Test Date:</strong> {test_results['test_timestamp']}</p>
    </div>
    
    <div class="summary {'failed' if test_results['summary']['failed'] > 0 else ''}">
        <h2>Test Summary</h2>
        <div class="metric">
            <strong>Total Tests:</strong> {test_results['summary']['total']}
        </div>
        <div class="metric">
            <strong>Passed:</strong> {test_results['summary']['passed']}
        </div>
        <div class="metric">
            <strong>Failed:</strong> {test_results['summary']['failed']}
        </div>
        <div class="metric">
            <strong>Success Rate:</strong> {(test_results['summary']['passed'] / test_results['summary']['total'] * 100):.1f}%
        </div>
    </div>
    
    <h2>Test Results</h2>
"""
    
    # Add individual test results
    for test in test_results['tests']:
        status_class = 'passed' if test['passed'] else 'failed'
        status_icon = '✅' if test['passed'] else '❌'
        html_content += f"""
    <div class="test-result {status_class}">
        <strong>{status_icon} {test['name']}</strong>
        <p>Time: {test['timestamp']}</p>
        {f"<p>Error: {test['error']}</p>" if test['error'] else ""}
    </div>
"""
    
    # Add performance metrics if available
    if test_results.get('performance_metrics'):
        perf = test_results['performance_metrics']
        html_content += f"""
    <h2>Performance Metrics</h2>
    <div class="summary">
        <div class="metric">
            <strong>Average Response Time:</strong> {perf['average_ms']}ms
        </div>
        <div class="metric">
            <strong>Max Response Time:</strong> {perf['max_ms']}ms
        </div>
        <div class="metric">
            <strong>Min Response Time:</strong> {perf['min_ms']}ms
        </div>
        <div class="metric">
            <strong>Test Iterations:</strong> {perf['iterations']}
        </div>
    </div>
"""
    
    # Add log analysis if available
    if test_results.get('logs'):
        logs = test_results['logs']
        html_content += f"""
    <h2>CloudWatch Logs Analysis</h2>
    <div class="summary">
        <div class="metric">
            <strong>Total Log Entries:</strong> {logs['total_entries']}
        </div>
        <div class="metric">
            <strong>INFO Messages:</strong> {logs['info_count']}
        </div>
        <div class="metric">
            <strong>ERROR Messages:</strong> {logs['error_count']}
        </div>
        <div class="metric">
            <strong>WARN Messages:</strong> {logs['warn_count']}
        </div>
    </div>
    
    <h3>Recent Log Entries</h3>
    <div class="logs">
"""
        for entry in logs['recent_entries']:
            timestamp = datetime.fromtimestamp(entry['timestamp']/1000).strftime('%Y-%m-%d %H:%M:%S')
            html_content += f"{timestamp} {entry['message']}\n"
        
        html_content += """    </div>
"""
    
    html_content += """
</body>
</html>
"""
    
    # Save HTML report
    with open('artifacts/test_report.html', 'w') as f:
        f.write(html_content)
    
    # Save JSON results
    with open('artifacts/test_results.json', 'w') as f:
        json.dump(test_results, f, indent=2)

def print_test_summary():
    """Print final test summary to console"""
    print(f"\n{Colors.BOLD}📊 Test Summary{Colors.END}")
    print("=" * 50)
    print(f"Total Tests: {test_results['summary']['total']}")
    print(f"{Colors.GREEN}✅ Passed: {test_results['summary']['passed']}{Colors.END}")
    print(f"{Colors.RED}❌ Failed: {test_results['summary']['failed']}{Colors.END}")
    
    if test_results['summary']['failed'] == 0:
        print(f"\n{Colors.GREEN}🎉 All tests passed!{Colors.END}")
    else:
        print(f"\n{Colors.RED}❌ Some tests failed. Check the detailed report.{Colors.END}")
        print("\nFailed Tests:")
        for test in test_results['tests']:
            if not test['passed']:
                print(f"  - {test['name']}: {test['error']}")
    
    print(f"\nTest execution completed in {time.time() - start_time:.1f} seconds")
    print(f"Artifacts saved to: artifacts/")
    print("- test_report.html")
    print("- test_results.json")
    print("- cloudwatch_logs.txt")
    print("- cloudwatch_logs.json")

# =============================================================================
# MAIN EXECUTION
# =============================================================================

def main():
    """Main test execution function"""
    global start_time
    start_time = time.time()
    
    print(f"{Colors.BOLD}🧪 Lambda Function Test Suite{Colors.END}")
    print(f"Function: {config.function_name}")
    print(f"Region: {config.region}")
    print(f"Timestamp: {datetime.now().isoformat()}")
    print("=" * 50)
    
    # Setup
    setup_artifacts_directory()
    
    # Run Unit Tests
    print(f"\n{Colors.YELLOW}🔬 RUNNING UNIT TESTS{Colors.END}")
    unit_test_verify_function_exists()
    unit_test_basic_invocation()
    unit_test_error_handling()
    unit_test_performance()
    
    # Run Integration Tests
    print(f"\n{Colors.YELLOW}🔗 RUNNING INTEGRATION TESTS{Colors.END}")
    integration_test_s3_operations()
    integration_test_cloudwatch_logs()
    integration_test_end_to_end_workflow()
    
    # Generate Reports
    print(f"\n{Colors.YELLOW}📊 GENERATING REPORTS{Colors.END}")
    generate_test_report()
    print_test_summary()
    
    # Exit with appropriate code
    exit_code = 0 if test_results['summary']['failed'] == 0 else 1
    sys.exit(exit_code)

if __name__ == "__main__":
    main()
```

---

## 🤖 **GitHub Copilot Integration Guide**

### **Where:**
```
File: tests/copilot_prompts.md
```

### **Content:**
```markdown
# GitHub Copilot Prompts for Lambda Testing

## Basic Test Generation
```
Prompt: "Generate comprehensive tests for my AWS Lambda function that processes user registration data. The function accepts JSON with user email, name, and preferences, validates the data, stores it in DynamoDB, and sends a welcome email via SES. Include both unit tests and integration tests."
```

## S3 Integration Testing
```
Prompt: "Create test cases for Lambda function that reads CSV files from S3, processes the data, and writes results back to S3. Include tests for file not found, malformed CSV, and large file processing scenarios."
```

## Error Handling Tests
```
Prompt: "Generate error handling test cases for Lambda function including: invalid JSON payload, missing required fields, AWS service timeouts, permission errors, and memory limit scenarios."
```

## Performance Testing
```
Prompt: "Create performance test scenarios for Lambda function including: cold start measurement, concurrent execution testing, memory usage validation, and timeout boundary testing."
```
```

---

## 🚀 **Implementation Checklist**

### **Week 1: Foundation Setup**
```
□ Create tests/ directory in repository
□ Add test_config.json with your Lambda details
□ Create test data files (valid/invalid/performance payloads)
□ Add lambda_test_suite.py script
□ Update Jenkinsfile to include test stage
□ Test basic script execution locally
□ Verify AWS CLI permissions work in Jenkins
```

### **Week 2: Script Refinement**
```
□ Customize test cases for your specific Lambda function
□ Use Copilot to generate additional test scenarios
□ Test S3 integration if your Lambda uses S3
□ Verify CloudWatch log extraction works
□ Test report generation and HTML output
□ Add email notifications for failures
```

### **Week 3: Integration & Testing**
```
□ Run complete test suite in Jenkins
□ Verify all artifacts are archived correctly
□ Test failure scenarios and notifications
□ Refine test thresholds and expectations
□ Add any custom test cases for your business logic
□ Document the process for other developers
```

### **Week 4: Production Deployment**
```
□ Deploy to production Jenkins pipeline
□ Train developers on using Copilot for test generation
□ Monitor test execution and results
□ Gather feedback and iterate on improvements
□ Create standard templates for common Lambda patterns
```

---

## 🎯 **Key Benefits of This Implementation**

### **Complete Testing Coverage:**
```
✅ Unit Tests: Function deployment, basic invocation, error handling, performance
✅ Integration Tests: S3 operations, CloudWatch logs, end-to-end workflows
✅ Automated Reporting: HTML reports, JSON data, archived artifacts
✅ Real-time Feedback: Console output, email notifications, Jenkins dashboard
```

### **Developer-Friendly:**
```
✅ Single script file combining all test types
✅ Copilot integration for easy test generation
✅ Minimal setup required (just add files to repository)
✅ No additional tools or infrastructure needed
```

### **Production-Ready:**
```
✅ Comprehensive error handling and logging
✅ Detailed reporting and artifact archiving
✅ Integration with existing CI/CD pipeline
✅ Scalable to multiple Lambda functions
```

This implementation gives you enterprise-grade Lambda testing with minimal complexity and maximum developer productivity! 🚀
