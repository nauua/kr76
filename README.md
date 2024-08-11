# KR76 - 2FA POC

This project contains the backend system for generating and validating single-use tokens using AWS services.

## Prerequisites

- AWS CLI installed and configured with the necessary permissions.
- Bash shell (e.g., Git Bash, WSL) for running the script on Windows.
- AWS CloudFormation.

## Steps to Set Up the Project

### Clone the Repository

Clone this repository to your local machine:

```bash
git clone https://github.com/nauua/kr76.git
```

Navigate to the project directory:

```bash
cd kr76
```

### Run the Upload Script

Run the provided bash script to create the S3 bucket and upload the necessary files:

```bash
./uploadfiles.sh
```

### Deploy the CloudFormation Stack

1. Go to the AWS Management Console and navigate to the CloudFormation service.
2. Click on Create Stack and choose With new resources (standard).
3. Select Upload a template file, choose the cloudformation.yml file from this repository, and click Next.
4. Follow the prompts and click Next until you reach the end, then click Create stack.

### Get API URLs from CloudFormation Outputs

After the stack creation is complete, go to the Outputs section of the CloudFormation stack.

Copy the provided URLs for the GenerateTokenAPI and ValidateTokenAPI.

### Test the APIs using Postman

1. Open Postman and create a new POST request.
2. For the Generate Token endpoint, use the copied URL for the GenerateTokenAPI and add the path /t1/generatetoken.
3. For the Validate Token endpoint, use the copied URL for the ValidateTokenAPI and add the path /t2/validatetoken.
4. Send requests to these endpoints to test the token generation and validation functionality.

## Architecture Diagram
![2FA Architecture](2fa-architecture.jpeg)

## Architecture Components

1. **API Gateway**: To expose endpoints for requesting and verifying tokens.
2. **Lambda Functions**: To handle the logic for generating and verifying tokens.
3. **DynamoDB**: To store tokens and their expiration times.
4. **SNS (Simple Notification Service)**: To send tokens via SMS or email.
5. **CloudWatch**: For logging, monitoring, and metrics.

## Code Excerpts

### 1. API Gateway and Lambda Integration
Create an API Gateway with two endpoints: `/request-token` and `/verify-token`.

### 2. Lambda Function to Generate Token
```python:src/backend/generate_token.py
import os
import boto3
import random
import string
from datetime import datetime, timedelta

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])

def generate_token(event, context):
    user_id = event['user_id']
    token = ''.join(random.choices(string.digits, k=6))
    expiration_time = datetime.utcnow() + timedelta(minutes=5)
    
    table.put_item(
        Item={
            'user_id': user_id,
            'token': token,
            'expires_at': expiration_time.isoformat()
        }
    )
    
    sns.publish(
        PhoneNumber=event['phone_number'],
        Message=f'Your verification code is {token}'
    )
    
    return {
        'statusCode': 200,
        'body': 'Token sent successfully'
    }
```

### 3. Lambda Function to Verify Token
```python:src/backend/verify_token.py
import os
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])

def verify_token(event, context):
    user_id = event['user_id']
    token = event['token']
    
    response = table.get_item(
        Key={'user_id': user_id}
    )
    
    if 'Item' not in response:
        return {'statusCode': 400, 'body': 'Invalid token'}
    
    item = response['Item']
    if item['token'] == token and datetime.fromisoformat(item['expires_at']) > datetime.utcnow():
        return {'statusCode': 200, 'body': 'Token verified successfully'}
    else:
        return {'statusCode': 400, 'body': 'Invalid or expired token'}
```

## Configuration Details
- **DynamoDB Table**: Create a table with `user_id` as the partition key.
- **Environment Variables**: Set `DYNAMODB_TABLE` in Lambda environment variables.

## Health/Availability & Performance Metrics
- **CloudWatch Alarms**: Set up alarms for Lambda errors and DynamoDB read/write capacity.
- **CloudWatch Logs**: Enable detailed logging for Lambda functions.

## Security Best Practices
- **IAM Roles**: Use least privilege principle for Lambda execution roles.
- **Encryption**: Enable encryption at rest for DynamoDB and SNS.
- **Secrets Management**: Use AWS Secrets Manager for sensitive information.