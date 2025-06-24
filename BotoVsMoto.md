Boto3 vs Moto - Key Differences

Boto3 (AWS SDK)

What it is: The official Python library for interacting with AWS services
Purpose: Makes actual API calls to AWS services (DynamoDB, S3, Lambda, etc.)
Your Lambda functions use this: When you write import boto3 and client = boto3.client('dynamodb'), you're using boto3
Real AWS connection: Sends HTTP requests to actual AWS endpoints and returns real data
Always required: Your Lambda functions need boto3 to talk to AWS services
Moto (Mocking Library)

What it is: A Python library that pretends to be AWS services for testing
Purpose: Intercepts boto3 calls and returns fake responses instead of calling real AWS
Testing tool only: Used during unit testing to avoid AWS costs and dependencies
Fake AWS responses: When boto3 tries to read from DynamoDB, moto returns fake data you define
No real AWS: Nothing actually gets sent to AWS services
Which One We're Using

In Your Lambda Functions: Always boto3

import boto3
dynamodb = boto3.client('dynamodb')  # This is boto3


In Jenkins Testing Pipeline: Both boto3 AND moto

Your Lambda code still uses boto3 (no changes needed)
Moto intercepts the boto3 calls during testing
boto3 thinks it's talking to real AWS, but moto gives it fake responses
Testing Strategy Summary

Unit Testing: boto3 + moto (fake AWS responses)
Integration Testing: boto3 + LocalStack (local AWS services)
QA Testing: boto3 + real AWS QA environment
UAT Testing: boto3 + real AWS UAT environment
Production: boto3 + real AWS production environment

Key Point: You only write code using boto3. Moto is just a testing helper that tricks boto3 into thinking it's getting real AWS responses when it's actually getting fake test data. Your Lambda function code never changes - it always uses boto3.

Think of it like this:

boto3 = your phone
AWS = the real person you're calling
moto = a recording that plays fake responses during testing so you don't actually call the real person
