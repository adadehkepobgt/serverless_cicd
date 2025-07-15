# 🐍 **Python Implementation for Single Lambda Testing**

Here's the complete Python implementation that's cleaner and more robust than the bash version:

---

## **📁 Python Project Structure**

```
your-project/
├── tests/
│   ├── lambda_tester.py          # Main testing class
│   ├── test_runner.py            # Test execution script
│   ├── config/
│   │   ├── project-config.json   # Basic project settings
│   │   ├── unit-tests.json       # Unit tests for your Lambda
│   │   └── integration-tests.json # Integration tests
│   ├── events/
│   │   └── test-events.json      # Test payloads
│   └── reports/                  # Generated test reports
├── wrapper/                      # Your existing Lambda code
└── Jenkinsfile                   # Clean pipeline
```

---

## **🔧 Simplified Jenkinsfile (Python Version)**

```groovy
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        S3_BUCKET = 'ss-dev-eucm-cicd-apnortheast1-bucket'
        TFE_WORKSPACE_ID = 'ws-AvXmDpoz88DscM5K'
        TFE_BASE_URL = 'https://tfe.dev.useast1.aws.jefco.com/api/v2'
        IAC_REPO_URL = 'https://bitbucket.corp.jefco.com/scm/euc/aws-ss-eucm-poc.git'
        BITBUCKET_API_URL = 'https://bitbucket.corp.jefco.com/rest/api/1.0'
        IAC_BRANCH = 'main'
        WORKSPACE_ID = 'EUC'
        REPO_SLUG = 'euc-lambda-poc'
        
        // Testing environment
        BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'unknown'}"
        TEST_RESULTS_DIR = 'tests/reports'
    }

    stages {
        stage('Checkout') {
            steps {
                echo "🔍 Checking out branch..."
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()
                }
            }
        }

        stage('Build Lambda Package') {
            steps {
                echo "📦 Creating Lambda deployment package..."
                script {
                    zip zipFile: 'lambda_deployment.zip', archive: false, dir: 'wrapper'
                }
            }
        }

        stage('Generate S3 Key') {
            steps {
                script {
                    def timestamp = sh(returnStdout: true, script: 'date "+%Y%m%d-%H%M%S"').trim()
                    env.S3_KEY = "lambda_functions/dev-${timestamp}-${env.GIT_COMMIT_SHORT}-lambda.zip"
                }
            }
        }

        stage('Upload to S3') {
            when {
                expression { env.GIT_BRANCH == 'origin/dev' }
            }
            steps {
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    s3Upload(
                        bucket: "${S3_BUCKET}",
                        file: "lambda_deployment.zip",
                        path: "${S3_KEY}"
                    )
                }
            }
        }

        stage('Checkout IaC Repository') {
            steps {
                git(
                    url: "${IAC_REPO_URL}",
                    branch: "${IAC_BRANCH}",
                    credentialsId: 'Bitbucket TEST'
                )
            }
        }

        stage('Update S3 Key in main.tf') {
            steps {
                sh """
                    sed -i 's|s3_key.*=.*|s3_key = \\"${S3_KEY}\\"|' ./dev/main.tf
                    sed -i 's|depends_on_object.*=.*|depends_on_object = \\"${S3_KEY}\\"|' ./dev/main.tf
                """
            }
        }

        stage('Commit and Push Changes') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'Bitbucket TEST', gitToolName: 'Default')]) {
                    sh """
                        git config user.name "Hannah Cia"
                        git config user.email "hannah.cia@jefferies.com"
                        git add ./dev/main.tf
                        git commit -m "Updated s3_key for lambda module to ${S3_KEY}"
                        git push origin ${IAC_BRANCH}
                    """
                }
            }
        }

        stage('Wait for Terraform Deployment') {
            steps {
                echo "⏳ Waiting for Terraform deployment to complete..."
                sleep(60)
            }
        }

        // PYTHON TESTING - Clean and robust
        
        stage('Setup Testing Environment') {
            steps {
                echo "🔧 Setting up Python testing environment..."
                sh """
                    # Install Python dependencies if needed
                    pip3 install boto3 --user --quiet
                    
                    # Create test directories
                    mkdir -p ${TEST_RESULTS_DIR}/unit
                    mkdir -p ${TEST_RESULTS_DIR}/integration
                """
            }
        }

        stage('Unit Tests') {
            steps {
                echo "🧪 Running unit tests with Python..."
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh 'python3 tests/test_runner.py unit'
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: "${TEST_RESULTS_DIR}/unit/*.xml"
                    archiveArtifacts artifacts: "${TEST_RESULTS_DIR}/unit/**/*", allowEmptyArchive: true
                }
            }
        }

        stage('Integration Tests') {
            steps {
                echo "🔗 Running integration tests with Python..."
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh 'python3 tests/test_runner.py integration'
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: "${TEST_RESULTS_DIR}/integration/*.xml"
                    archiveArtifacts artifacts: "${TEST_RESULTS_DIR}/integration/**/*", allowEmptyArchive: true
                }
            }
        }

        stage('Merge Pull Request') {
            when {
                expression { env.BITBUCKET_PULL_REQUEST_ID != null }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Bitbucket TEST', usernameVariable: 'BITBUCKET_CREDENTIALS_USR', passwordVariable: 'BITBUCKET_CREDENTIALS_PSW')]) {
                    script {
                        def response = sh(
                            script: """
                                curl -X GET -u \${BITBUCKET_CREDENTIALS_USR}:\${BITBUCKET_CREDENTIALS_PSW} \\
                                    -H "Content-Type: application/json" \\
                                    "\${BITBUCKET_API_URL}/projects/\${WORKSPACE_ID}/repos/\${REPO_SLUG}/pull-requests/\${env.BITBUCKET_PULL_REQUEST_ID}"
                            """,
                            returnStdout: true
                        ).trim()

                        def jsonResponse = readJSON text: response
                        env.PULL_REQUEST_VERSION = jsonResponse.version

                        sh """
                            curl -X POST -u \${BITBUCKET_CREDENTIALS_USR}:\${BITBUCKET_CREDENTIALS_PSW} \\
                                -H "Content-Type: application/json" \\
                                -d '{"message": "Tests passed - auto-merged by Jenkins", "version": \${env.PULL_REQUEST_VERSION}}' \\
                                "\${BITBUCKET_API_URL}/projects/\${WORKSPACE_ID}/repos/\${REPO_SLUG}/pull-requests/\${env.BITBUCKET_PULL_REQUEST_ID}/merge"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${TEST_RESULTS_DIR}/**/*", allowEmptyArchive: true
        }
        success {
            echo '🎉 Pipeline completed successfully! Lambda deployed and tested.'
        }
        failure {
            echo '❌ Pipeline failed. Check test results for details.'
        }
    }
}
```

---

## **🐍 Python Testing Implementation**

### **`tests/lambda_tester.py` - Main Testing Class**

```python
#!/usr/bin/env python3
"""
Lambda Function Testing Framework
Simplified for single Lambda function testing
"""

import boto3
import json
import os
import sys
import time
from datetime import datetime
from typing import Dict, List, Any, Optional
import xml.etree.ElementTree as ET
from botocore.exceptions import ClientError, NoCredentialsError

class LambdaTester:
    """Main class for testing Lambda functions"""
    
    def __init__(self, region: str = 'ap-northeast-1'):
        """Initialize the Lambda tester"""
        self.region = region
        self.build_id = os.getenv('BUILD_ID', datetime.now().strftime('%Y%m%d-%H%M%S'))
        self.results_dir = os.getenv('TEST_RESULTS_DIR', 'tests/reports')
        
        # Initialize AWS clients
        try:
            self.lambda_client = boto3.client('lambda', region_name=region)
            print(f"✅ AWS Lambda client initialized for region: {region}")
        except NoCredentialsError:
            print("❌ AWS credentials not found")
            sys.exit(1)
        except Exception as e:
            print(f"❌ Failed to initialize AWS client: {e}")
            sys.exit(1)
    
    def find_lambda_function(self, pattern: str = 'euc-lambda-poc') -> str:
        """Find the deployed Lambda function"""
        print(f"🔍 Looking for Lambda function matching: {pattern}")
        
        try:
            response = self.lambda_client.list_functions()
            matching_functions = [
                func for func in response['Functions'] 
                if pattern in func['FunctionName']
            ]
            
            if not matching_functions:
                raise Exception(f"No Lambda function found matching '{pattern}'")
            
            function_name = matching_functions[0]['FunctionName']
            print(f"✅ Found Lambda function: {function_name}")
            
            # Save function name for other tests
            os.makedirs(self.results_dir, exist_ok=True)
            with open(f"{self.results_dir}/lambda-function-name.txt", 'w') as f:
                f.write(function_name)
            
            return function_name
            
        except ClientError as e:
            error_code = e.response['Error']['Code']
            error_message = e.response['Error']['Message']
            print(f"❌ AWS Error finding Lambda function: {error_code}: {error_message}")
            sys.exit(1)
        except Exception as e:
            print(f"❌ Error finding Lambda function: {e}")
            sys.exit(1)
    
    def verify_lambda_access(self, function_name: str) -> bool:
        """Verify we can access the Lambda function"""
        try:
            self.lambda_client.get_function(FunctionName=function_name)
            print(f"✅ Lambda function '{function_name}' is accessible")
            return True
        except ClientError as e:
            print(f"❌ Cannot access Lambda function '{function_name}': {e}")
            return False
    
    def invoke_lambda(self, function_name: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Invoke Lambda function and return structured response"""
        try:
            print(f"    🚀 Invoking Lambda function: {function_name}")
            
            start_time = time.time()
            response = self.lambda_client.invoke(
                FunctionName=function_name,
                Payload=json.dumps(payload)
            )
            duration = time.time() - start_time
            
            # Read the response payload
            response_payload = response['Payload'].read()
            result = json.loads(response_payload) if response_payload else {}
            
            return {
                'success': response['StatusCode'] == 200,
                'status_code': response.get('StatusCode', 0),
                'duration': round(duration * 1000, 2),  # in milliseconds
                'payload': result,
                'error': result.get('errorType') if isinstance(result, dict) and 'errorType' in result else None,
                'error_message': result.get('errorMessage') if isinstance(result, dict) else None
            }
            
        except ClientError as e:
            return {
                'success': False,
                'status_code': 0,
                'duration': 0,
                'payload': {},
                'error': e.response['Error']['Code'],
                'error_message': e.response['Error']['Message']
            }
        except Exception as e:
            return {
                'success': False,
                'status_code': 0,
                'duration': 0,
                'payload': {},
                'error': 'InvocationError',
                'error_message': str(e)
            }
    
    def validate_response(self, actual: Dict[str, Any], expected: Dict[str, Any]) -> tuple[bool, str]:
        """Validate Lambda response against expectations"""
        
        # Check status code
        if 'statusCode' in expected:
            actual_status = actual.get('payload', {}).get('statusCode', 200)
            expected_status = expected['statusCode']
            
            if actual_status != expected_status:
                return False, f"Status code mismatch: expected {expected_status}, got {actual_status}"
        
        # Check if response body contains specific text
        if 'bodyContains' in expected:
            response_body = str(actual.get('payload', {}))
            for required_text in expected['bodyContains']:
                if required_text not in response_body:
                    return False, f"Response missing required text: '{required_text}'"
        
        # Check for error types
        if 'errorType' in expected:
            actual_error = actual.get('error')
            expected_error = expected['errorType']
            
            if actual_error != expected_error:
                return False, f"Error type mismatch: expected {expected_error}, got {actual_error}"
        
        return True, "Validation passed"
    
    def load_config(self, config_file: str) -> Dict[str, Any]:
        """Load test configuration from JSON file"""
        config_path = f"tests/config/{config_file}"
        
        try:
            with open(config_path, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            print(f"⚠️ Configuration file not found: {config_path}")
            return self.create_default_config(config_file)
        except json.JSONDecodeError as e:
            print(f"❌ Invalid JSON in {config_path}: {e}")
            sys.exit(1)
    
    def create_default_config(self, config_file: str) -> Dict[str, Any]:
        """Create default configuration if file doesn't exist"""
        os.makedirs("tests/config", exist_ok=True)
        
        if config_file == "unit-tests.json":
            default_config = {
                "scenarios": [
                    {
                        "name": "Basic Health Check",
                        "event": {
                            "test": "health",
                            "source": "jenkins"
                        },
                        "expected": {
                            "statusCode": 200
                        }
                    },
                    {
                        "name": "Error Handling Test",
                        "event": {
                            "action": "invalid"
                        },
                        "expected": {
                            "statusCode": 400
                        }
                    }
                ]
            }
        elif config_file == "integration-tests.json":
            default_config = {
                "workflows": [
                    {
                        "name": "Basic Lambda Workflow",
                        "description": "Test basic Lambda functionality",
                        "steps": [
                            {
                                "name": "Test Lambda Function",
                                "type": "invoke_lambda",
                                "payload": {
                                    "test": "integration",
                                    "workflow": "basic",
                                    "timestamp": "${timestamp}"
                                },
                                "expect": {
                                    "statusCode": 200
                                }
                            }
                        ]
                    }
                ]
            }
        else:
            default_config = {}
        
        # Save default config
        config_path = f"tests/config/{config_file}"
        with open(config_path, 'w') as f:
            json.dump(default_config, f, indent=2)
        
        print(f"📝 Created default configuration: {config_path}")
        return default_config
    
    def run_unit_tests(self) -> bool:
        """Run unit tests for the Lambda function"""
        print("🧪 Running Unit Tests")
        print("=" * 50)
        
        # Find Lambda function
        function_name = self.find_lambda_function()
        
        # Verify access
        if not self.verify_lambda_access(function_name):
            return False
        
        # Load test configuration
        config = self.load_config("unit-tests.json")
        
        print(f"Testing Lambda: {function_name}")
        print(f"Region: {self.region}")
        print(f"Build ID: {self.build_id}")
        print()
        
        # Run test scenarios
        results = []
        
        for scenario in config.get('scenarios', []):
            scenario_name = scenario['name']
            test_event = scenario['event']
            expected = scenario['expected']
            
            print(f"  📝 Scenario: {scenario_name}")
            
            # Invoke Lambda
            response = self.invoke_lambda(function_name, test_event)
            
            # Save response for debugging
            self.save_response(response, f"unit/{scenario_name.replace(' ', '-')}")
            
            # Validate response
            if response['success'] and not response['error']:
                is_valid, validation_msg = self.validate_response(response, expected)
                
                if is_valid:
                    print(f"    ✅ PASSED: {scenario_name} ({response['duration']}ms)")
                    results.append({
                        'name': scenario_name,
                        'passed': True,
                        'duration': response['duration'],
                        'error': None
                    })
                else:
                    print(f"    ❌ FAILED: {scenario_name} - {validation_msg}")
                    results.append({
                        'name': scenario_name,
                        'passed': False,
                        'duration': response['duration'],
                        'error': validation_msg
                    })
            else:
                error_msg = response['error_message'] or response['error'] or "Unknown error"
                print(f"    ❌ FAILED: {scenario_name} - {error_msg}")
                results.append({
                    'name': scenario_name,
                    'passed': False,
                    'duration': response['duration'],
                    'error': error_msg
                })
        
        # Generate reports
        self.generate_junit_report(results, 'unit', function_name)
        self.generate_summary_report(results, 'unit', function_name)
        
        # Print summary
        passed = sum(1 for r in results if r['passed'])
        total = len(results)
        avg_duration = sum(r['duration'] for r in results) / len(results) if results else 0
        
        print()
        print("📊 Unit Test Summary:")
        print("=" * 30)
        print(f"Total: {total}")
        print(f"Passed: {passed}")
        print(f"Failed: {total - passed}")
        print(f"Average Duration: {avg_duration:.2f}ms")
        print(f"Lambda Function: {function_name}")
        
        return passed == total
    
    def run_integration_tests(self) -> bool:
        """Run integration tests for Lambda workflows"""
        print("🔗 Running Integration Tests")
        print("=" * 50)
        
        # Find Lambda function
        function_name = self.find_lambda_function()
        
        # Verify access
        if not self.verify_lambda_access(function_name):
            return False
        
        # Load test configuration
        config = self.load_config("integration-tests.json")
        
        print(f"Testing Lambda: {function_name}")
        print(f"Build ID: {self.build_id}")
        print()
        
        # Run workflows
        workflow_results = []
        
        for workflow in config.get('workflows', []):
            workflow_name = workflow['name']
            workflow_description = workflow.get('description', '')
            
            print(f"  🔗 Workflow: {workflow_name}")
            if workflow_description:
                print(f"      Description: {workflow_description}")
            
            workflow_success = True
            workflow_steps = []
            
            # Execute workflow steps
            for step_num, step in enumerate(workflow.get('steps', []), 1):
                step_name = step['name']
                step_type = step['type']
                
                print(f"    Step {step_num}: {step_name}")
                
                step_result = self.execute_workflow_step(step, function_name, step_num)
                workflow_steps.append(step_result)
                
                if not step_result['success']:
                    workflow_success = False
                    print(f"      ❌ Step failed: {step_result['error']}")
                else:
                    print(f"      ✅ Step passed ({step_result['duration']}ms)")
            
            # Record workflow result
            if workflow_success:
                print(f"  ✅ WORKFLOW PASSED: {workflow_name}")
            else:
                print(f"  ❌ WORKFLOW FAILED: {workflow_name}")
            
            workflow_results.append({
                'name': workflow_name,
                'passed': workflow_success,
                'steps': workflow_steps
            })
        
        # Generate reports
        self.generate_integration_junit_report(workflow_results, function_name)
        self.generate_integration_summary_report(workflow_results, function_name)
        
        # Print summary
        passed_workflows = sum(1 for w in workflow_results if w['passed'])
        total_workflows = len(workflow_results)
        
        print()
        print("📊 Integration Test Summary:")
        print("=" * 35)
        print(f"Total Workflows: {total_workflows}")
        print(f"Passed: {passed_workflows}")
        print(f"Failed: {total_workflows - passed_workflows}")
        print(f"Lambda Function: {function_name}")
        
        return passed_workflows == total_workflows
    
    def execute_workflow_step(self, step: Dict[str, Any], function_name: str, step_num: int) -> Dict[str, Any]:
        """Execute a single workflow step"""
        step_type = step['type']
        
        if step_type == 'invoke_lambda':
            payload = step['payload'].copy()
            
            # Replace timestamp placeholder
            if isinstance(payload, dict):
                payload = self.replace_placeholders(payload)
            
            # Invoke Lambda
            response = self.invoke_lambda(function_name, payload)
            
            # Save response
            step_name = step['name'].replace(' ', '-')
            self.save_response(response, f"integration/{step_name}-step-{step_num}")
            
            # Validate expectations if provided
            if 'expect' in step:
                is_valid, validation_msg = self.validate_response(response, step['expect'])
                
                return {
                    'success': response['success'] and is_valid,
                    'duration': response['duration'],
                    'error': None if response['success'] and is_valid else (validation_msg or response['error_message'])
                }
            else:
                return {
                    'success': response['success'],
                    'duration': response['duration'],
                    'error': response['error_message'] if not response['success'] else None
                }
        
        elif step_type == 'wait':
            wait_seconds = step.get('seconds', 5)
            print(f"      ⏳ Waiting {wait_seconds} seconds...")
            time.sleep(wait_seconds)
            
            return {
                'success': True,
                'duration': wait_seconds * 1000,  # Convert to ms for consistency
                'error': None
            }
        
        else:
            return {
                'success': False,
                'duration': 0,
                'error': f"Unknown step type: {step_type}"
            }
    
    def replace_placeholders(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Replace placeholders in payload with actual values"""
        payload_str = json.dumps(payload)
        
        # Replace timestamp
        payload_str = payload_str.replace('${timestamp}', datetime.now().isoformat())
        payload_str = payload_str.replace('${build_id}', self.build_id)
        
        return json.loads(payload_str)
    
    def save_response(self, response: Dict[str, Any], filename: str):
        """Save Lambda response to file for debugging"""
        os.makedirs(f"{self.results_dir}/{filename.split('/')[0]}", exist_ok=True)
        
        response_file = f"{self.results_dir}/{filename}-{self.build_id}.json"
        
        with open(response_file, 'w') as f:
            json.dump({
                'timestamp': datetime.now().isoformat(),
                'build_id': self.build_id,
                'response': response
            }, f, indent=2)
    
    def generate_junit_report(self, results: List[Dict], test_type: str, function_name: str):
        """Generate JUnit XML report for Jenkins"""
        os.makedirs(f"{self.results_dir}/{test_type}", exist_ok=True)
        
        total_tests = len(results)
        failed_tests = sum(1 for r in results if not r['passed'])
        total_time = sum(r['duration'] for r in results) / 1000  # Convert to seconds
        
        testsuite = ET.Element('testsuite', {
            'name': f'{function_name} {test_type.title()} Tests',
            'tests': str(total_tests),
            'failures': str(failed_tests),
            'time': f"{total_time:.3f}",
            'timestamp': datetime.now().isoformat()
        })
        
        # Add properties
        properties = ET.SubElement(testsuite, 'properties')
        ET.SubElement(properties, 'property', name='lambda.function', value=function_name)
        ET.SubElement(properties, 'property', name='aws.region', value=self.region)
        ET.SubElement(properties, 'property', name='build.id', value=self.build_id)
        
        # Add test cases
        for result in results:
            testcase = ET.SubElement(testsuite, 'testcase', {
                'name': result['name'],
                'time': f"{result['duration'] / 1000:.3f}"
            })
            
            if not result['passed']:
                failure = ET.SubElement(testcase, 'failure', message=result['error'] or 'Test failed')
                failure.text = result['error'] or 'Unknown error'
        
        # Write XML file
        tree = ET.ElementTree(testsuite)
        ET.indent(tree, space="  ", level=0)
        tree.write(f"{self.results_dir}/{test_type}/junit-results.xml", encoding='utf-8', xml_declaration=True)
    
    def generate_integration_junit_report(self, workflow_results: List[Dict], function_name: str):
        """Generate JUnit XML report for integration tests"""
        os.makedirs(f"{self.results_dir}/integration", exist_ok=True)
        
        total_workflows = len(workflow_results)
        failed_workflows = sum(1 for w in workflow_results if not w['passed'])
        
        testsuite = ET.Element('testsuite', {
            'name': f'{function_name} Integration Tests',
            'tests': str(total_workflows),
            'failures': str(failed_workflows),
            'time': '0',
            'timestamp': datetime.now().isoformat()
        })
        
        # Add properties
        properties = ET.SubElement(testsuite, 'properties')
        ET.SubElement(properties, 'property', name='lambda.function', value=function_name)
        ET.SubElement(properties, 'property', name='aws.region', value=self.region)
        ET.SubElement(properties, 'property', name='build.id', value=self.build_id)
        
        # Add workflow test cases
        for workflow in workflow_results:
            testcase = ET.SubElement(testsuite, 'testcase', {
                'name': workflow['name'],
                'time': '0'
            })
            
            if not workflow['passed']:
                failure = ET.SubElement(testcase, 'failure', message='Workflow failed')
                failure.text = f"Workflow '{workflow['name']}' failed"
        
        # Write XML file
        tree = ET.ElementTree(testsuite)
        ET.indent(tree, space="  ", level=0)
        tree.write(f"{self.results_dir}/integration/junit-results.xml", encoding='utf-8', xml_declaration=True)
    
    def generate_summary_report(self, results: List[Dict], test_type: str, function_name: str):
        """Generate JSON summary report"""
        summary = {
            'timestamp': datetime.now().isoformat(),
            'build_id': self.build_id,
            'lambda_function': function_name,
            'test_type': test_type,
            'total': len(results),
            'passed': sum(1 for r in results if r['passed']),
            'failed': sum(1 for r in results if not r['passed']),
            'average_duration_ms': sum(r['duration'] for r in results) / len(results) if results else 0,
            'results': results
        }
        
        with open(f"{self.results_dir}/{test_type}/summary.json", 'w') as f:
            json.dump(summary, f, indent=2)
    
    def generate_integration_summary_report(self, workflow_results: List[Dict], function_name: str):
        """Generate JSON summary report for integration tests"""
        summary = {
            'timestamp': datetime.now().isoformat(),
            'build_id': self.build_id,
            'lambda_function': function_name,
            'test_type': 'integration',
            'total_workflows': len(workflow_results),
            'passed_workflows': sum(1 for w in workflow_results if w['passed']),
            'failed_workflows': sum(1 for w in workflow_results if not w['passed']),
            'workflows': workflow_results
        }
        
        with open(f"{self.results_dir}/integration/summary.json", 'w') as f:
            json.dump(summary, f, indent=2)
```

### **`tests/test_runner.py` - Test Execution Script**

```python
#!/usr/bin/env python3
"""
Test Runner Script
Entry point for running Lambda tests
"""

import sys
import os
from lambda_tester import LambdaTester

def main():
    """Main entry point for test execution"""
    
    if len(sys.argv) < 2:
        print("Usage: python3 test_runner.py [unit|integration|all]")
        print()
        print("Commands:")
        print("  unit        - Run unit tests only")
        print("  integration - Run integration tests only")
        print("  all         - Run all tests")
        sys.exit(1)
    
    test_type = sys.argv[1].lower()
    
    # Get configuration from environment
    region = os.getenv('AWS_DEFAULT_REGION', 'ap-northeast-1')
    
    # Initialize tester
    print(f"🚀 Initializing Lambda Tester")
    print(f"Region: {region}")
    print(f"Test Type: {test_type}")
    print()
    
    tester = LambdaTester(region=region)
    
    success = True
    
    try:
        if test_type == 'unit':
            success = tester.run_unit_tests()
            
        elif test_type == 'integration':
            success = tester.run_integration_tests()
            
        elif test_type == 'all':
            print("🧪 Running All Tests")
            print("=" * 50)
            
            unit_success = tester.run_unit_tests()
            print("\n" + "=" * 50 + "\n")
            integration_success = tester.run_integration_tests()
            
            success = unit_success and integration_success
            
            print("\n" + "=" * 50)
            print("📊 Overall Test Summary:")
            print(f"Unit Tests: {'✅ PASSED' if unit_success else '❌ FAILED'}")
            print(f"Integration Tests: {'✅ PASSED' if integration_success else '❌ FAILED'}")
            print(f"Overall: {'✅ ALL TESTS PASSED' if success else '❌ SOME TESTS FAILED'}")
            
        else:
            print(f"❌ Unknown test type: {test_type}")
            print("Valid options: unit, integration, all")
            sys.exit(1)
    
    except KeyboardInterrupt:
        print("\n❌ Tests interrupted by user")
        sys.exit(1)
    except Exception as e:
        print(f"\n❌ Unexpected error during testing: {e}")
        sys.exit(1)
    
    # Exit with appropriate code
    if success:
        print(f"\n🎉 All {test_type} tests passed!")
        sys.exit(0)
    else:
        print(f"\n❌ Some {test_type} tests failed!")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## **📋 Configuration Files (Same as before but cleaner)**

### **`tests/config/unit-tests.json`**
```json
{
  "scenarios": [
    {
      "name": "Basic Health Check",
      "event": {
        "test": "health",
        "source": "jenkins"
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Valid Input Test",
      "event": {
        "action": "process",
        "data": {
          "message": "test data"
        }
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["success"]
      }
    },
    {
      "name": "Error Handling Test",
      "event": {
        "action": "invalid"
      },
      "expected": {
        "statusCode": 400
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
      "name": "Basic Lambda Workflow",
      "description": "Test basic Lambda functionality with multiple steps",
      "steps": [
        {
          "name": "Initialize Process",
          "type": "invoke_lambda",
          "payload": {
            "action": "initialize",
            "timestamp": "${timestamp}",
            "build_id": "${build_id}"
          },
          "expect": {
            "statusCode": 200
          }
        },
        {
          "name": "Wait for Processing",
          "type": "wait",
          "seconds": 2
        },
        {
          "name": "Verify Result",
          "type": "invoke_lambda",
          "payload": {
            "action": "verify",
            "test": "integration"
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

## **🎯 Key Advantages of Python Implementation**

### **✅ Better Error Handling:**
```python
# Clear, structured error handling
try:
    response = self.lambda_client.invoke(FunctionName=function_name, Payload=payload)
    return self.process_response(response)
except ClientError as e:
    return self.handle_aws_error(e)
except Exception as e:
    return self.handle_general_error(e)
```

### **✅ Type Safety & IntelliSense:**
```python
def invoke_lambda(self, function_name: str, payload: Dict[str, Any]) -> Dict[str, Any]:
    # Clear types help with development and debugging
```

### **✅ Native JSON Handling:**
```python
# No jq gymnastics needed
test_event = scenario['event']
expected_status = expected.get('statusCode', 200)
```

### **✅ Rich Reporting:**
```python
# Detailed JUnit XML with proper structure
# JSON summary reports with all test data
# Response files saved for debugging
```

### **✅ Easy to Extend:**
```python
# Add new step types easily
def execute_workflow_step(self, step):
    if step_type == 'invoke_lambda':
        return self.invoke_lambda_step(step)
    elif step_type == 'verify_database':
        return self.verify_database_step(step)  # Easy to add
    elif step_type == 'send_sqs':
        return self.send_sqs_step(step)  # Easy to add
```

---

## **🚀 Usage**

### **Run Tests Locally:**
```bash
# Unit tests only
python3 tests/test_runner.py unit

# Integration tests only
python3 tests/test_runner.py integration

# All tests
python3 tests/test_runner.py all
```

### **Jenkins Integration:**
```groovy
# Clean and simple
sh 'python3 tests/test_runner.py unit'
sh 'python3 tests/test_runner.py integration'
```

---

## **📊 Output Examples**

### **Unit Test Output:**
```
🧪 Running Unit Tests
==================================================
🔍 Looking for Lambda function matching: euc-lambda-poc
✅ Found Lambda function: euc-lambda-poc-dev
✅ Lambda function 'euc-lambda-poc-dev' is accessible
Testing Lambda: euc-lambda-poc-dev
Region: ap-northeast-1
Build ID: 123-a1b2c3d

  📝 Scenario: Basic Health Check
    🚀 Invoking Lambda function: euc-lambda-poc-dev
    ✅ PASSED: Basic Health Check (245.67ms)

  📝 Scenario: Valid Input Test
    🚀 Invoking Lambda function: euc-lambda-poc-dev
    ✅ PASSED: Valid Input Test (198.23ms)

📊 Unit Test Summary:
==============================
Total: 2
Passed: 2
Failed: 0
Average Duration: 221.95ms
Lambda Function: euc-lambda-poc-dev

🎉 All unit tests passed!
```

**This Python implementation is much more robust, maintainable, and feature-rich than the bash version!** 🐍🎯

# 🎯 **Updated Python Testing Framework for RDS-Connected Lambda**

Looking at your Lambda function, I can see it handles database operations with different endpoints. Let me adjust the testing framework to properly test your RDS-connected Lambda.

---

## **📋 Updated Configuration Files**

### **`tests/config/unit-tests.json` - RDS Lambda Tests**
```json
{
  "scenarios": [
    {
      "name": "RetrieveTaskById - Valid Request",
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
      "name": "RetrieveTaskById - Missing Parameters",
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
      "name": "RetrieveDistinctColumns - Valid Request",
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
      "name": "RetrieveJobList - Valid Request",
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
      "name": "RetrieveView - Valid Request",
      "event": {
        "operation": "RetrieveView",
        "queryStringParameters": {
          "table_name": "test_view",
          "role_tag": "admin",
          "role_merge": "view_all"
        }
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "RetrieveAuditTrail - Valid Request",
      "event": {
        "operation": "RetrieveAuditTrail",
        "queryStringParameters": {
          "start_date": "2024-01-01",
          "end_date": "2024-12-31"
        }
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Submit - Valid Data",
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
      "name": "Update - Valid Data",
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
      "name": "Invalid Operation",
      "event": {
        "operation": "InvalidOperation"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["error", "Invalid operation"]
      }
    },
    {
      "name": "Health Check",
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

### **`tests/config/integration-tests.json` - RDS Workflow Tests**
```json
{
  "workflows": [
    {
      "name": "Complete Task Management Workflow",
      "description": "Test full task lifecycle: create, retrieve, update, audit",
      "steps": [
        {
          "name": "Submit New Task",
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
          "name": "Wait for Database Commit",
          "type": "wait",
          "seconds": 2
        },
        {
          "name": "Retrieve Task by ID",
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
          "name": "Update Task Status",
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
          "name": "Wait for Update",
          "type": "wait",
          "seconds": 1
        },
        {
          "name": "Verify Updated Task",
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
      "name": "Database Query Operations Workflow",
      "description": "Test various database query operations",
      "steps": [
        {
          "name": "Get Job List",
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
          "name": "Get Distinct Column Values",
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
        },
        {
          "name": "Get Filtered View",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveView",
            "queryStringParameters": {
              "table_name": "test_view",
              "role_tag": "developer",
              "role_merge": "view_limited"
            }
          },
          "expect": {
            "statusCode": 200
          }
        }
      ]
    },
    {
      "name": "Audit Trail Workflow",
      "description": "Test audit trail functionality",
      "steps": [
        {
          "name": "Submit Task for Audit",
          "type": "invoke_lambda",
          "payload": {
            "operation": "Submit",
            "queryStringParameters": {
              "table_name": "test_tasks"
            },
            "body": "{\"task_id\": \"audit_test_${timestamp}\", \"status\": \"pending\", \"description\": \"Audit test task\"}"
          },
          "expect": {
            "statusCode": 200
          }
        },
        {
          "name": "Wait for Audit Log",
          "type": "wait",
          "seconds": 3
        },
        {
          "name": "Retrieve Audit Trail",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveAuditTrail",
            "queryStringParameters": {
              "start_date": "${today}",
              "end_date": "${today}"
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

## **🔧 Enhanced Lambda Tester for RDS Operations**

### **`tests/lambda_tester.py` - Updated for RDS Testing**

```python
#!/usr/bin/env python3
"""
Lambda Function Testing Framework for RDS-Connected Lambda
Specifically designed for database operations testing
"""

import boto3
import json
import os
import sys
import time
import uuid
from datetime import datetime, date
from typing import Dict, List, Any, Optional
import xml.etree.ElementTree as ET
from botocore.exceptions import ClientError, NoCredentialsError

class RDSLambdaTester:
    """Enhanced Lambda tester for RDS-connected functions"""
    
    def __init__(self, region: str = 'ap-northeast-1'):
        """Initialize the RDS Lambda tester"""
        self.region = region
        self.build_id = os.getenv('BUILD_ID', datetime.now().strftime('%Y%m%d-%H%M%S'))
        self.results_dir = os.getenv('TEST_RESULTS_DIR', 'tests/reports')
        
        # Generate unique test session ID
        self.test_session_id = f"test_{self.build_id}_{uuid.uuid4().hex[:8]}"
        
        # Initialize AWS clients
        try:
            self.lambda_client = boto3.client('lambda', region_name=region)
            print(f"✅ AWS Lambda client initialized for region: {region}")
            print(f"🔧 Test session ID: {self.test_session_id}")
        except NoCredentialsError:
            print("❌ AWS credentials not found")
            sys.exit(1)
        except Exception as e:
            print(f"❌ Failed to initialize AWS client: {e}")
            sys.exit(1)
    
    def find_lambda_function(self, pattern: str = 'euc-lambda-poc') -> str:
        """Find the deployed Lambda function"""
        print(f"🔍 Looking for Lambda function matching: {pattern}")
        
        try:
            response = self.lambda_client.list_functions()
            matching_functions = [
                func for func in response['Functions'] 
                if pattern in func['FunctionName']
            ]
            
            if not matching_functions:
                raise Exception(f"No Lambda function found matching '{pattern}'")
            
            function_name = matching_functions[0]['FunctionName']
            print(f"✅ Found Lambda function: {function_name}")
            
            # Save function info
            os.makedirs(self.results_dir, exist_ok=True)
            function_info = {
                'name': function_name,
                'runtime': matching_functions[0].get('Runtime', 'unknown'),
                'timeout': matching_functions[0].get('Timeout', 0),
                'memory': matching_functions[0].get('MemorySize', 0),
                'last_modified': matching_functions[0].get('LastModified', '')
            }
            
            with open(f"{self.results_dir}/lambda-function-info.json", 'w') as f:
                json.dump(function_info, f, indent=2)
            
            return function_name
            
        except ClientError as e:
            print(f"❌ AWS Error: {e.response['Error']['Code']}: {e.response['Error']['Message']}")
            sys.exit(1)
        except Exception as e:
            print(f"❌ Error finding Lambda function: {e}")
            sys.exit(1)
    
    def verify_lambda_access(self, function_name: str) -> bool:
        """Verify we can access the Lambda function and check basic info"""
        try:
            response = self.lambda_client.get_function(FunctionName=function_name)
            config = response['Configuration']
            
            print(f"✅ Lambda function '{function_name}' is accessible")
            print(f"   Runtime: {config.get('Runtime', 'unknown')}")
            print(f"   Timeout: {config.get('Timeout', 0)}s")
            print(f"   Memory: {config.get('MemorySize', 0)}MB")
            
            # Check if function has environment variables (might indicate RDS config)
            env_vars = config.get('Environment', {}).get('Variables', {})
            if env_vars:
                print(f"   Environment variables: {len(env_vars)} configured")
                # Don't print actual values for security
                env_keys = list(env_vars.keys())
                db_related = [key for key in env_keys if any(term in key.lower() for term in ['db', 'database', 'rds', 'host', 'endpoint'])]
                if db_related:
                    print(f"   Database-related env vars detected: {len(db_related)}")
            
            return True
            
        except ClientError as e:
            print(f"❌ Cannot access Lambda function '{function_name}': {e}")
            return False
    
    def invoke_lambda(self, function_name: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Invoke Lambda function with enhanced response handling for RDS operations"""
        try:
            print(f"    🚀 Invoking Lambda: {function_name}")
            
            # Log the operation being tested
            operation = payload.get('operation', 'unknown')
            print(f"       Operation: {operation}")
            
            start_time = time.time()
            response = self.lambda_client.invoke(
                FunctionName=function_name,
                Payload=json.dumps(payload)
            )
            duration = time.time() - start_time
            
            # Read the response payload
            response_payload = response['Payload'].read()
            result = json.loads(response_payload) if response_payload else {}
            
            # Enhanced response analysis for RDS operations
            response_analysis = self.analyze_rds_response(result, operation)
            
            return {
                'success': response['StatusCode'] == 200,
                'status_code': response.get('StatusCode', 0),
                'duration': round(duration * 1000, 2),  # in milliseconds
                'payload': result,
                'error': result.get('errorType') if isinstance(result, dict) and 'errorType' in result else None,
                'error_message': result.get('errorMessage') if isinstance(result, dict) else None,
                'operation': operation,
                'analysis': response_analysis
            }
            
        except ClientError as e:
            return {
                'success': False,
                'status_code': 0,
                'duration': 0,
                'payload': {},
                'error': e.response['Error']['Code'],
                'error_message': e.response['Error']['Message'],
                'operation': payload.get('operation', 'unknown'),
                'analysis': {'db_connected': False, 'data_returned': False}
            }
        except Exception as e:
            return {
                'success': False,
                'status_code': 0,
                'duration': 0,
                'payload': {},
                'error': 'InvocationError',
                'error_message': str(e),
                'operation': payload.get('operation', 'unknown'),
                'analysis': {'db_connected': False, 'data_returned': False}
            }
    
    def analyze_rds_response(self, result: Dict[str, Any], operation: str) -> Dict[str, Any]:
        """Analyze Lambda response for RDS-specific indicators"""
        analysis = {
            'db_connected': False,
            'data_returned': False,
            'record_count': 0,
            'has_error': False,
            'response_type': 'unknown'
        }
        
        if not isinstance(result, dict):
            return analysis
        
        # Check for successful database connection indicators
        body_str = str(result.get('body', ''))
        
        # Successful operation indicators
        if any(indicator in body_str.lower() for indicator in ['success', 'task_data', 'submitted successfully']):
            analysis['db_connected'] = True
            analysis['response_type'] = 'success'
        
        # Error indicators
        if any(error_indicator in body_str.lower() for error_indicator in ['error', 'failed', 'exception']):
            analysis['has_error'] = True
            analysis['response_type'] = 'error'
        
        # Data indicators
        try:
            if 'body' in result:
                body_content = json.loads(result['body']) if isinstance(result['body'], str) else result['body']
                
                # Check for data structures
                if isinstance(body_content, list):
                    analysis['data_returned'] = True
                    analysis['record_count'] = len(body_content)
                elif isinstance(body_content, dict):
                    analysis['data_returned'] = True
                    # Check for common data patterns
                    if 'task_data' in body_content or 'results' in body_content:
                        analysis['record_count'] = 1
        except:
            pass
        
        # Operation-specific analysis
        if operation in ['RetrieveTaskById', 'RetrieveJobList', 'RetrieveView', 'RetrieveAuditTrail']:
            analysis['db_connected'] = not analysis['has_error']
        elif operation in ['Submit', 'Update']:
            analysis['db_connected'] = 'success' in body_str.lower()
        
        return analysis
    
    def validate_response(self, actual: Dict[str, Any], expected: Dict[str, Any]) -> tuple[bool, str]:
        """Enhanced validation for RDS Lambda responses"""
        
        # Standard status code validation
        if 'statusCode' in expected:
            actual_status = actual.get('payload', {}).get('statusCode', 200)
            expected_status = expected['statusCode']
            
            if actual_status != expected_status:
                return False, f"Status code mismatch: expected {expected_status}, got {actual_status}"
        
        # Body content validation
        if 'bodyContains' in expected:
            response_body = str(actual.get('payload', {}))
            for required_text in expected['bodyContains']:
                if required_text not in response_body:
                    return False, f"Response missing required text: '{required_text}'"
        
        # RDS-specific validations
        if 'requiresDbConnection' in expected and expected['requiresDbConnection']:
            if not actual.get('analysis', {}).get('db_connected', False):
                return False, "Database connection required but not detected"
        
        if 'expectsData' in expected and expected['expectsData']:
            if not actual.get('analysis', {}).get('data_returned', False):
                return False, "Data expected but not returned"
        
        if 'minRecords' in expected:
            record_count = actual.get('analysis', {}).get('record_count', 0)
            min_records = expected['minRecords']
            if record_count < min_records:
                return False, f"Expected at least {min_records} records, got {record_count}"
        
        return True, "Validation passed"
    
    def replace_placeholders(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Enhanced placeholder replacement for RDS testing"""
        payload_str = json.dumps(payload)
        
        # Standard replacements
        current_time = datetime.now()
        payload_str = payload_str.replace('${timestamp}', current_time.strftime('%Y%m%d_%H%M%S'))
        payload_str = payload_str.replace('${build_id}', self.build_id)
        payload_str = payload_str.replace('${test_session}', self.test_session_id)
        
        # Date replacements
        today = date.today()
        payload_str = payload_str.replace('${today}', today.strftime('%Y-%m-%d'))
        payload_str = payload_str.replace('${yesterday}', (today - timedelta(days=1)).strftime('%Y-%m-%d'))
        
        # Unique ID replacements
        payload_str = payload_str.replace('${uuid}', str(uuid.uuid4()))
        payload_str = payload_str.replace('${short_uuid}', str(uuid.uuid4())[:8])
        
        return json.loads(payload_str)
    
    def run_unit_tests(self) -> bool:
        """Run unit tests specifically designed for RDS Lambda operations"""
        print("🧪 Running RDS Lambda Unit Tests")
        print("=" * 50)
        
        # Find Lambda function
        function_name = self.find_lambda_function()
        
        # Verify access
        if not self.verify_lambda_access(function_name):
            return False
        
        # Load test configuration
        config = self.load_config("unit-tests.json")
        
        print(f"Testing Lambda: {function_name}")
        print(f"Region: {self.region}")
        print(f"Test Session: {self.test_session_id}")
        print()
        
        # Run test scenarios
        results = []
        
        for scenario in config.get('scenarios', []):
            scenario_name = scenario['name']
            test_event = scenario['event']
            expected = scenario['expected']
            
            print(f"  📝 Scenario: {scenario_name}")
            
            # Invoke Lambda
            response = self.invoke_lambda(function_name, test_event)
            
            # Save detailed response for debugging
            self.save_detailed_response(response, f"unit/{scenario_name.replace(' ', '-')}")
            
            # Validate response
            if response['success'] and not response['error']:
                is_valid, validation_msg = self.validate_response(response, expected)
                
                if is_valid:
                    operation = response.get('operation', 'unknown')
                    analysis = response.get('analysis', {})
                    db_status = "📊 DB Connected" if analysis.get('db_connected') else "⚠️ DB Issue"
                    
                    print(f"    ✅ PASSED: {scenario_name} ({response['duration']}ms) {db_status}")
                    
                    results.append({
                        'name': scenario_name,
                        'passed': True,
                        'duration': response['duration'],
                        'error': None,
                        'operation': operation,
                        'db_connected': analysis.get('db_connected', False),
                        'data_returned': analysis.get('data_returned', False)
                    })
                else:
                    print(f"    ❌ FAILED: {scenario_name} - {validation_msg}")
                    results.append({
                        'name': scenario_name,
                        'passed': False,
                        'duration': response['duration'],
                        'error': validation_msg,
                        'operation': response.get('operation', 'unknown'),
                        'db_connected': False,
                        'data_returned': False
                    })
            else:
                error_msg = response['error_message'] or response['error'] or "Unknown error"
                print(f"    ❌ FAILED: {scenario_name} - {error_msg}")
                results.append({
                    'name': scenario_name,
                    'passed': False,
                    'duration': response['duration'],
                    'error': error_msg,
                    'operation': response.get('operation', 'unknown'),
                    'db_connected': False,
                    'data_returned': False
                })
        
        # Generate enhanced reports
        self.generate_rds_junit_report(results, 'unit', function_name)
        self.generate_rds_summary_report(results, 'unit', function_name)
        
        # Print enhanced summary
        passed = sum(1 for r in results if r['passed'])
        total = len(results)
        avg_duration = sum(r['duration'] for r in results) / len(results) if results else 0
        db_connected_count = sum(1 for r in results if r.get('db_connected', False))
        
        print()
        print("📊 RDS Lambda Unit Test Summary:")
        print("=" * 40)
        print(f"Total: {total}")
        print(f"Passed: {passed}")
        print(f"Failed: {total - passed}")
        print(f"Average Duration: {avg_duration:.2f}ms")
        print(f"DB Operations Successful: {db_connected_count}/{total}")
        print(f"Lambda Function: {function_name}")
        
        return passed == total
    
    def save_detailed_response(self, response: Dict[str, Any], filename: str):
        """Save detailed Lambda response including RDS analysis"""
        os.makedirs(f"{self.results_dir}/{filename.split('/')[0]}", exist_ok=True)
        
        response_file = f"{self.results_dir}/{filename}-{self.build_id}.json"
        
        detailed_response = {
            'timestamp': datetime.now().isoformat(),
            'build_id': self.build_id,
            'test_session': self.test_session_id,
            'lambda_response': response,
            'rds_analysis': response.get('analysis', {}),
            'test_metadata': {
                'operation': response.get('operation', 'unknown'),
                'success': response.get('success', False),
                'duration_ms': response.get('duration', 0),
                'db_connected': response.get('analysis', {}).get('db_connected', False)
            }
        }
        
        with open(response_file, 'w') as f:
            json.dump(detailed_response, f, indent=2)
    
    def generate_rds_junit_report(self, results: List[Dict], test_type: str, function_name: str):
        """Generate enhanced JUnit XML report with RDS-specific information"""
        os.makedirs(f"{self.results_dir}/{test_type}", exist_ok=True)
        
        total_tests = len(results)
        failed_tests = sum(1 for r in results if not r['passed'])
        total_time = sum(r['duration'] for r in results) / 1000
        
        testsuite = ET.Element('testsuite', {
            'name': f'{function_name} RDS {test_type.title()} Tests',
            'tests': str(total_tests),
            'failures': str(failed_tests),
            'time': f"{total_time:.3f}",
            'timestamp': datetime.now().isoformat()
        })
        
        # Enhanced properties
        properties = ET.SubElement(testsuite, 'properties')
        ET.SubElement(properties, 'property', name='lambda.function', value=function_name)
        ET.SubElement(properties, 'property', name='aws.region', value=self.region)
        ET.SubElement(properties, 'property', name='build.id', value=self.build_id)
        ET.SubElement(properties, 'property', name='test.session', value=self.test_session_id)
        ET.SubElement(properties, 'property', name='db.operations.successful', 
                     value=str(sum(1 for r in results if r.get('db_connected', False))))
        
        # Add test cases with RDS information
        for result in results:
            testcase_attrs = {
                'name': result['name'],
                'time': f"{result['duration'] / 1000:.3f}",
                'classname': f"RDS.{result.get('operation', 'Unknown')}"
            }
            
            testcase = ET.SubElement(testsuite, 'testcase', testcase_attrs)
            
            # Add RDS-specific properties to each test case
            test_properties = ET.SubElement(testcase, 'properties')
            ET.SubElement(test_properties, 'property', name='operation', 
                         value=result.get('operation', 'unknown'))
            ET.SubElement(test_properties, 'property', name='db.connected', 
                         value=str(result.get('db_connected', False)))
            ET.SubElement(test_properties, 'property', name='data.returned', 
                         value=str(result.get('data_returned', False)))
            
            if not result['passed']:
                failure = ET.SubElement(testcase, 'failure', 
                                      message=result['error'] or 'Test failed')
                failure.text = f"Operation: {result.get('operation', 'unknown')}\nError: {result['error'] or 'Unknown error'}"
        
        # Write XML file
        tree = ET.ElementTree(testsuite)
        ET.indent(tree, space="  ", level=0)
        tree.write(f"{self.results_dir}/{test_type}/junit-results.xml", 
                  encoding='utf-8', xml_declaration=True)
    
    def generate_rds_summary_report(self, results: List[Dict], test_type: str, function_name: str):
        """Generate enhanced JSON summary report with RDS metrics"""
        db_operations = {}
        for result in results:
            operation = result.get('operation', 'unknown')
            if operation not in db_operations:
                db_operations[operation] = {'total': 0, 'passed': 0, 'db_connected': 0}
            
            db_operations[operation]['total'] += 1
            if result['passed']:
                db_operations[operation]['passed'] += 1
            if result.get('db_connected', False):
                db_operations[operation]['db_connected'] += 1
        
        summary = {
            'timestamp': datetime.now().isoformat(),
            'build_id': self.build_id,
            'test_session': self.test_session_id,
            'lambda_function': function_name,
            'test_type': test_type,
            'total': len(results),
            'passed': sum(1 for r in results if r['passed']),
            'failed': sum(1 for r in results if not r['passed']),
            'average_duration_ms': sum(r['duration'] for r in results) / len(results) if results else 0,
            'db_operations_summary': db_operations,
            'db_connectivity': {
                'successful_connections': sum(1 for r in results if r.get('db_connected', False)),
                'total_db_operations': len([r for r in results if r.get('operation', '') != 'unknown']),
                'connection_rate': sum(1 for r in results if r.get('db_connected', False)) / len(results) if results else 0
            },
            'results': results
        }
        
        with open(f"{self.results_dir}/{test_type}/summary.json", 'w') as f:
            json.dump(summary, f, indent=2)
    
    # Include all other methods from the previous implementation...
    # (load_config, create_default_config, run_integration_tests, etc.)
    
    def load_config(self, config_file: str) -> Dict[str, Any]:
        """Load test configuration from JSON file"""
        config_path = f"tests/config/{config_file}"
        
        try:
            with open(config_path, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            print(f"⚠️ Configuration file not found: {config_path}")
            return self.create_default_config(config_file)
        except json.JSONDecodeError as e:
            print(f"❌ Invalid JSON in {config_path}: {e}")
            sys.exit(1)
    
    def create_default_config(self, config_file: str) -> Dict[str, Any]:
        """Create default configuration for RDS Lambda testing"""
        os.makedirs("tests/config", exist_ok=True)
        
        if config_file == "unit-tests.json":
            # Use the RDS-specific config from above
            default_config = {
                "scenarios": [
                    {
                        "name": "Basic Health Check",
                        "event": {"test": "health", "source": "jenkins"},
                        "expected": {"statusCode": 200}
                    }
                ]
            }
        elif config_file == "integration-tests.json":
            # Use the RDS workflow config from above  
            default_config = {
                "workflows": [
                    {
                        "name": "Basic RDS Test",
                        "steps": [
                            {
                                "name": "Test Database Connection",
                                "type": "invoke_lambda",
                                "payload": {"operation": "RetrieveJobList", "queryStringParameters": {"table_name": "test_table"}},
                                "expect": {"statusCode": 200}
                            }
                        ]
                    }
                ]
            }
        
        config_path = f"tests/config/{config_file}"
        with open(config_path, 'w') as f:
            json.dump(default_config, f, indent=2)
        
        print(f"📝 Created default RDS configuration: {config_path}")
        return default_config
```

### **`tests/test_runner.py` - Updated Entry Point**

```python
#!/usr/bin/env python3
"""
RDS Lambda Test Runner
Entry point for running RDS-connected Lambda tests
"""

import sys
import os
from datetime import timedelta
from lambda_tester import RDSLambdaTester

def main():
    """Main entry point for RDS Lambda test execution"""
    
    if len(sys.argv) < 2:
        print("Usage: python3 test_runner.py [unit|integration|all]")
        print()
        print("Commands:")
        print("  unit        - Run unit tests for RDS operations")
        print("  integration - Run integration workflow tests")
        print("  all         - Run all RDS Lambda tests")
        sys.exit(1)
    
    test_type = sys.argv[1].lower()
    
    # Get configuration from environment
    region = os.getenv('AWS_DEFAULT_REGION', 'ap-northeast-1')
    
    # Initialize RDS Lambda tester
    print(f"🚀 Initializing RDS Lambda Tester")
    print(f"Region: {region}")
    print(f"Test Type: {test_type}")
    print("=" * 50)
    
    tester = RDSLambdaTester(region=region)
    
    success = True
    
    try:
        if test_type == 'unit':
            success = tester.run_unit_tests()
            
        elif test_type == 'integration':
            success = tester.run_integration_tests()
            
        elif test_type == 'all':
            print("🧪 Running All RDS Lambda Tests")
            print("=" * 50)
            
            unit_success = tester.run_unit_tests()
            print("\n" + "=" * 50 + "\n")
            integration_success = tester.run_integration_tests()
            
            success = unit_success and integration_success
            
            print("\n" + "=" * 50)
            print("📊 Overall RDS Lambda Test Summary:")
            print(f"Unit Tests: {'✅ PASSED' if unit_success else '❌ FAILED'}")
            print(f"Integration Tests: {'✅ PASSED' if integration_success else '❌ FAILED'}")
            print(f"Overall: {'✅ ALL TESTS PASSED' if success else '❌ SOME TESTS FAILED'}")
            
        else:
            print(f"❌ Unknown test type: {test_type}")
            print("Valid options: unit, integration, all")
            sys.exit(1)
    
    except KeyboardInterrupt:
        print("\n❌ Tests interrupted by user")
        sys.exit(1)
    except Exception as e:
        print(f"\n❌ Unexpected error during RDS Lambda testing: {e}")
        import traceback
        traceback.print_exc()
        sys.exit(1)
    
    # Exit with appropriate code
    if success:
        print(f"\n🎉 All {test_type} tests passed! RDS Lambda is working correctly.")
        sys.exit(0)
    else:
        print(f"\n❌ Some {test_type} tests failed! Check RDS connectivity and Lambda logic.")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## **🎯 Key Enhancements for RDS Testing**

### **✅ RDS-Specific Validations:**
- **Database Connection Detection:** Analyzes responses to determine if DB operations succeeded
- **Data Return Validation:** Checks if queries returned expected data
- **Operation-Specific Testing:** Different validation logic for CRUD operations

### **✅ Enhanced Error Analysis:**
- **DB Connection Issues:** Detects database connectivity problems
- **Query Failures:** Identifies SQL or data access errors
- **Response Structure Analysis:** Validates expected JSON response formats

### **✅ Better Test Data:**
- **Realistic Payloads:** Test events match your Lambda's expected input format
- **Multiple Operations:** Tests all your Lambda operations (Retrieve, Submit, Update, etc.)
- **Edge Cases:** Invalid operations, missing parameters, etc.

### **✅ Improved Reporting:**
- **DB Metrics:** Track database operation success rates
- **Performance Tracking:** Monitor query response times
- **Operation Breakdown:** Success rates per database operation type

**This enhanced framework specifically tests your RDS-connected Lambda with realistic database operations and provides detailed insights into both Lambda and database performance!** 🎯📊

# 🔧 **Prerequisites for RDS Lambda Testing Framework**

Here's everything you need to set up before running the testing framework:

---

## **🐍 Python Requirements**

### **1. Python Version**
```bash
# Check your Python version (needs 3.7+)
python3 --version

# If you don't have Python 3.7+, install it
# On Amazon Linux/CentOS:
sudo yum install python3 python3-pip

# On Ubuntu/Debian:
sudo apt-get install python3 python3-pip
```

### **2. Required Python Packages**
```bash
# Install boto3 (AWS SDK for Python)
pip3 install boto3 --user

# Verify installation
python3 -c "import boto3; print('boto3 version:', boto3.__version__)"
```

**Note:** The framework only uses Python standard library + boto3, so no other packages needed!

---

## **☁️ AWS Prerequisites**

### **3. AWS CLI (Already have this)**
```bash
# Verify AWS CLI is working
aws --version
aws sts get-caller-identity

# Should show your current AWS identity
```

### **4. AWS Permissions**
Your Jenkins role needs these permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:ListFunctions",
        "lambda:GetFunction",
        "lambda:InvokeFunction"
      ],
      "Resource": "*"
    }
  ]
}
```

**✅ You already have this through your existing Jenkins role: `arn:aws:iam::623341402407:role/jef-tfe-role`**

---

## **📁 File Structure Setup**

### **5. Create Directory Structure**
```bash
# From your project root, create the testing structure
mkdir -p tests/{config,events,reports}
mkdir -p tests/reports/{unit,integration}

# Verify structure
tree tests/
# Should show:
# tests/
# ├── config/
# ├── events/
# ├── reports/
# │   ├── integration/
# │   └── unit/
# ├── lambda_tester.py
# └── test_runner.py
```

---

## **🛠️ Jenkins Environment**

### **6. Jenkins Pipeline Environment Variables**
Make sure these are set in your Jenkinsfile (you already have most):
```groovy
environment {
    AWS_DEFAULT_REGION = 'ap-northeast-1'                    // ✅ You have this
    BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"  // ✅ You have this
    TEST_RESULTS_DIR = 'tests/reports'                       // Add this line
    
    // Optional: Set test configuration
    LAMBDA_FUNCTION_PATTERN = 'euc-lambda-poc'               // Add this line
}
```

### **7. Jenkins Plugins (should already be installed)**
- **AWS Steps Plugin** ✅ (you're using `withAWS`)
- **JUnit Plugin** ✅ (for `publishTestResults`)
- **Pipeline Plugin** ✅ (for pipeline syntax)

---

## **🗄️ RDS/Database Prerequisites**

### **8. Test Database Tables**
Your Lambda connects to RDS, so you need test tables. Create these in your **dev database**:

```sql
-- Example test tables (adjust to match your actual schema)
CREATE TABLE IF NOT EXISTS test_tasks (
    task_id VARCHAR(100) PRIMARY KEY,
    status VARCHAR(50),
    description TEXT,
    created_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS test_jobs (
    job_id VARCHAR(100) PRIMARY KEY,
    job_name VARCHAR(200),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert some test data
INSERT IGNORE INTO test_tasks (task_id, status, description, created_by)
VALUES 
    ('task_123', 'pending', 'Test task for CI/CD', 'system'),
    ('task_456', 'completed', 'Another test task', 'system');

INSERT IGNORE INTO test_jobs (job_id, job_name, status)
VALUES 
    ('job_001', 'Test Job 1', 'active'),
    ('job_002', 'Test Job 2', 'inactive');
```

### **9. Lambda Environment Variables**
Make sure your Lambda has these environment variables configured:
- Database connection settings (host, username, password, database name)
- Any other configuration your Lambda needs

**✅ This should already be configured since your Lambda is working**

---

## **📋 Quick Setup Script**

Create this setup script to verify everything is ready:

### **`tests/setup_prerequisites.sh`**
```bash
#!/bin/bash
echo "🔧 Checking Prerequisites for RDS Lambda Testing"
echo "================================================"

# Check Python
echo "1. Checking Python..."
if python3 --version >/dev/null 2>&1; then
    echo "   ✅ Python3 available: $(python3 --version)"
else
    echo "   ❌ Python3 not found"
    exit 1
fi

# Check boto3
echo "2. Checking boto3..."
if python3 -c "import boto3" >/dev/null 2>&1; then
    echo "   ✅ boto3 available: $(python3 -c "import boto3; print(boto3.__version__)")"
else
    echo "   ❌ boto3 not found. Install with: pip3 install boto3 --user"
    exit 1
fi

# Check AWS CLI
echo "3. Checking AWS CLI..."
if aws --version >/dev/null 2>&1; then
    echo "   ✅ AWS CLI available: $(aws --version)"
else
    echo "   ❌ AWS CLI not found"
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
for dir in "tests" "tests/config" "tests/reports" "tests/reports/unit" "tests/reports/integration"; do
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
    
    # Test basic access
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
echo "You're ready to run the RDS Lambda testing framework."
echo ""
echo "Quick test commands:"
echo "  python3 tests/test_runner.py unit"
echo "  python3 tests/test_runner.py integration"
```

---

## **🚀 Final Checklist**

### **Before First Run:**

**✅ Environment Setup:**
- [ ] Python 3.7+ installed
- [ ] boto3 package installed (`pip3 install boto3 --user`)
- [ ] AWS CLI working (`aws sts get-caller-identity`)

**✅ Files in Place:**
- [ ] `tests/lambda_tester.py` - Main testing class
- [ ] `tests/test_runner.py` - Test execution script
- [ ] `tests/config/unit-tests.json` - Unit test configuration
- [ ] `tests/config/integration-tests.json` - Integration test configuration

**✅ Jenkins Configuration:**
- [ ] `TEST_RESULTS_DIR = 'tests/reports'` added to environment
- [ ] Pipeline stages call `python3 tests/test_runner.py unit`

**✅ Database Setup:**
- [ ] Test tables exist in your dev RDS instance
- [ ] Lambda can connect to RDS (should already work)
- [ ] Test data available for querying

### **Quick Verification:**
```bash
# Run the prerequisite check
chmod +x tests/setup_prerequisites.sh
./tests/setup_prerequisites.sh

# If all checks pass, run a quick test
python3 tests/test_runner.py unit
```

---

## **📞 Troubleshooting Common Issues**

### **If boto3 is missing:**
```bash
pip3 install boto3 --user
# or if pip3 doesn't work:
python3 -m pip install boto3 --user
```

### **If AWS credentials don't work in Jenkins:**
Your existing `withAWS` block should handle this:
```groovy
withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
    sh 'python3 tests/test_runner.py unit'
}
```

### **If Lambda function not found:**
Check the function name pattern in the test configuration or set it explicitly:
```bash
export LAMBDA_FUNCTION_PATTERN="your-exact-function-name"
```

**That's it! You should be ready to run the RDS Lambda testing framework.** 🎯
