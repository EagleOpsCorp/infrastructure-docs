# Comprehensive AWS and DevOps Scenarios with Expanded HTTP Error Cases

This document provides detailed, real-world scenarios and solutions for AWS and DevOps concepts, tailored for operational troubleshooting and technical interviews. It covers CDN + Route 53, CloudWatch + CloudTrail, HTTP errors, AWS storage and networking, Dockerfiles, Go/Python DSA interview questions, mTLS optimization, and ECS vs EKS. Point 3 is expanded with additional HTTP error cases (4xx and 5xx), designed as interview-style problems. Point 5 includes multiple Dockerfile examples, Point 7 provides thorough mTLS details, and Point 8 offers detailed ECS vs EKS analysis. The Gunicorn-based Dockerfile incorporates WSGI and IPv6 (`[::]:80`).

---

## 1. CDN + Route 53

**Amazon CloudFront (CDN)**:
- **Purpose**: Reduces latency by caching content at edge locations.
- **Features**: Supports static/dynamic content, HTTPS, AWS Shield, Lambda@Edge.
- **Real-World Scenario**: E-commerce site needs low-latency image delivery.
  - **Solution**: CloudFront with S3 origin, geo-restrictions, Lambda@Edge for URL rewriting.
  - **Interview Tip**: Discuss cache invalidation (`aws cloudfront create-invalidation`).

**Route 53**:
- **Purpose**: Scalable DNS with routing policies (Weighted, Latency-based, Failover).
- **Real-World Scenario**: Multi-region app requires failover.
  - **Solution**: Route 53 Failover with primary ALB in us-east-1, secondary in us-west-2, health checks.
  - **Interview Tip**: Explain Alias records for CloudFront/ALB.

**Integration**: Route 53 resolves domains to CloudFront, caching ALB/S3 content.

---

## 2. CloudWatch Deployment + CloudTrail

**CloudWatch**:
- **Purpose**: Monitors resources via metrics, logs, alarms.
- **Real-World Scenario**: CI/CD pipeline deploys microservice with performance issues.
  - **Solution**: Track CPU/memory with CloudWatch Metrics, alarms for latency, Logs Insights for debugging.
  - **Interview Tip**: Highlight custom metrics.

**CloudTrail**:
- **Purpose**: Audits API calls for compliance.
- **Real-World Scenario**: Audit ECS service modifications.
  - **Solution**: Enable CloudTrail, store logs in S3, use Insights for anomalies.
  - **Interview Tip**: Discuss multi-region trails.

**Integration**: CloudWatch monitors deployments, CloudTrail logs API actions.

---

## 3. HTTP Errors in AWS: Real-Time Interview Problems

This section presents a comprehensive set of real-time, interview-style problems focusing on HTTP errors (4xx client-side and 5xx server-side) in AWS environments. Each problem simulates a production issue, requiring troubleshooting and resolution, mimicking challenges faced in operational roles and technical interviews. New cases have been added to provide a broader range of scenarios, covering additional AWS services and error conditions.

### Problem 1: Intermittent 403 Forbidden Errors in CloudFront (Client-Side)
**Context**: A media streaming platform uses CloudFront with an S3 origin for video content. After a recent deployment, users in specific regions report intermittent 403 Forbidden errors when accessing videos, while others experience no issues.
**Challenge**: Identify and resolve the 403 errors without disrupting global access.
**Troubleshooting**:
1. Check CloudFront’s geo-restriction settings in the AWS Console to rule out regional blocks.
2. Verify the S3 bucket policy to ensure it grants `s3:GetObject` to CloudFront’s Origin Access Identity (OAI).
3. Review CloudWatch Logs for CloudFront to identify error patterns (e.g., affected regions, specific files).
4. Validate S3 object permissions and metadata (e.g., public access or OAI).
5. Use `curl` from affected regions to reproduce the issue.
**Root Cause**: The S3 bucket policy was updated, inadvertently removing the OAI’s `s3:GetObject` permission for certain objects.
**Solution**:
- Update the S3 bucket policy:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E123456789"
              },
              "Action": "s3:GetObject",
              "Resource": "arn:aws:s3:::media-bucket/*"
          }
      ]
  }
  ```
- Ensure CloudFront distribution uses the correct OAI:
  ```bash
  aws cloudfront update-distribution --id <distribution-id> --distribution-config file://config.json
  ```
- Verify S3 object permissions to allow OAI access.
- Test access from affected regions.
- Set CloudWatch alarm for `4xxErrorRate > 1%`.
**Interview Tips**:
- Explain OAI’s role in securing S3 access.
- Discuss debugging with CloudWatch Logs and geo-restriction pitfalls.
- Highlight proactive monitoring with alarms.

### Problem 2: 502 Bad Gateway Errors in ALB After Deployment (Server-Side)
**Context**: A web application on EC2 instances behind an Application Load Balancer (ALB) returns 502 Bad Gateway errors after a new deployment. ALB health checks show some instances as unhealthy, impacting user experience.
**Challenge**: Diagnose the 502 errors and restore service without rolling back the deployment.
**Troubleshooting**:
1. Check ALB target group health in the AWS Console.
2. Review CloudWatch Logs for EC2 application errors (e.g., crashes, invalid responses).
3. Inspect ALB access logs for 502 error details (e.g., target response codes).
4. Verify application port and health check endpoint (e.g., `/health`).
5. SSH into an unhealthy instance and test locally (`curl http://localhost:80/health`).
**Root Cause**: The new application version misconfigures HTTP response headers (e.g., missing `Content-Length`), causing ALB to reject responses.
**Solution**:
- Fix application code (e.g., Node.js):
  ```javascript
  app.get('/health', (req, res) => {
      res.setHeader('Content-Type', 'application/json');
      res.status(200).send({ status: 'healthy' });
  });
  ```
- Update ALB health check to target `/health` (HTTP 200).
- Restart application on affected instances:
  ```bash
  sudo systemctl restart my-app
  ```
- Scale Auto Scaling Group to replace unhealthy instances:
  ```bash
  aws autoscaling update-auto-scaling-group --auto-scaling-group-name my-asg --desired-capacity 4
  ```
- Monitor CloudWatch `HTTPCode_ELB_5XX` with alarm for `> 0`.
**Interview Tips**:
- Explain ALB’s response validation requirements.
- Discuss health check misconfigurations as a common 502 cause.
- Highlight scaling and monitoring strategies.

### Problem 3: 429 Too Many Requests in API Gateway During Traffic Spike (Client-Side)
**Context**: An e-commerce platform’s checkout API, hosted on API Gateway with a Lambda backend, returns 429 Too Many Requests errors during a flash sale, causing failed transactions. A custom usage plan exists for premium customers, but all users are affected.
**Challenge**: Mitigate 429 errors to ensure smooth checkout.
**Troubleshooting**:
1. Check CloudWatch Metrics for API Gateway throttling (`Throttle` metric).
2. Review usage plan and account-level throttling limits (default: 10,000 requests/second).
3. Analyze Lambda logs for performance bottlenecks.
4. Inspect client request patterns (e.g., retry storms) using CloudWatch Logs Insights.
5. Test API with Postman to simulate traffic.
**Root Cause**: The usage plan’s rate limit (e.g., 1,000 requests/second) is too low for the spike, and account-level throttling is hit.
**Solution**:
- Increase usage plan limits:
  ```bash
  aws apigateway update-usage-plan --usage-plan-id <plan-id> --patch-operations op=replace,path=/throttle/rateLimit,value=2000 op=replace,path=/throttle/burstLimit,value=4000
  ```
- Request AWS Support to raise account-level throttling.
- Optimize Lambda:
  - Increase memory (e.g., 512 MB):
    ```bash
    aws lambda update-function-configuration --function-name my-checkout --memory-size 512
    ```
  - Enable provisioned concurrency:
    ```bash
    aws lambda put-provisioned-concurrency-config --function-name my-checkout --qualifier prod --provisioned-concurrent-executions 100
    ```
- Add CloudFront to cache GET responses:
  ```bash
  aws cloudfront create-distribution --distribution-config file://config.json
  ```
- Implement client-side exponential backoff for retries.
- Monitor CloudWatch `Throttle` and `4xxErrorRate` with alarms.
**Interview Tips**:
- Differentiate usage plans vs account-level throttling.
- Highlight caching and backoff strategies.
- Discuss Lambda concurrency limits.

### Problem 4: 400 Bad Request in API Gateway Due to Payload Issues (Client-Side) [New]
**Context**: A mobile application integrates with an API Gateway endpoint to submit user profile updates. After adding a new feature to upload profile pictures, some users receive 400 Bad Request errors when submitting data, disrupting the user experience.
**Challenge**: Identify the cause of the 400 errors and ensure reliable profile updates.
**Troubleshooting**:
1. Check CloudWatch Logs for API Gateway to identify malformed requests.
2. Review the API Gateway request validation settings in the AWS Console.
3. Inspect client-side code to validate payload structure (e.g., JSON, multipart form).
4. Reproduce the issue using Postman with sample payloads.
5. Verify API Gateway model schemas for request body validation.
**Root Cause**: The client sends malformed multipart form data (e.g., incorrect boundary headers) when uploading images, which API Gateway rejects due to strict request validation.
**Solution**:
- Update client-side code to correctly format multipart form data (e.g., using `requests` in Python):
  ```python
  import requests
  url = "https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/profile"
  files = {'picture': open('profile.jpg', 'rb')}
  data = {'name': 'John Doe'}
  response = requests.post(url, files=files, data=data)
  ```
- Enable request validation in API Gateway to provide detailed error messages:
  ```bash
  aws apigateway update-method --rest-api-id <api-id> --resource-id <resource-id> \
      --http-method POST --patch-operations op=replace,path=/requestValidatorId,value=<validator-id>
  ```
- Define a model schema for the request body:
  ```json
  {
      "type": "object",
      "properties": {
          "name": {"type": "string"},
          "picture": {"type": "string", "format": "binary"}
      },
      "required": ["name"]
  }
  ```
- Log detailed errors in CloudWatch for debugging:
  ```bash
  aws apigateway update-stage --rest-api-id <api-id> --stage-name prod \
      --patch-operations op=replace,path=/accessLogSettings/destinationArn,value=<log-group-arn>
  ```
- Test updated client code to confirm resolution.
- Set CloudWatch alarm for `4xxErrorRate > 5%`.
**Interview Tips**:
- Explain API Gateway’s request validation and model schemas.
- Discuss debugging malformed payloads with CloudWatch.
- Highlight client-side improvements for robust integrations.

### Problem 5: 500 Internal Server Error in Lambda with DynamoDB (Server-Side) [New]
**Context**: A serverless application uses API Gateway with a Lambda function to retrieve user data from DynamoDB. After updating the Lambda function to add a new query, users intermittently receive 500 Internal Server Error responses, causing data retrieval failures.
**Challenge**: Diagnose and fix the 500 errors to restore reliable data access.
**Troubleshooting**:
1. Check CloudWatch Logs for the Lambda function to identify errors (e.g., exceptions, timeouts).
2. Review DynamoDB metrics in CloudWatch for throttling or errors (e.g., `ThrottledRequests`).
3. Inspect the Lambda function code for query issues (e.g., incorrect keys, missing permissions).
4. Verify IAM role permissions for DynamoDB access.
5. Test the query manually using the AWS CLI:
  ```bash
  aws dynamodb query --table-name Users --key-condition-expression "userId = :id" \
      --expression-attribute-values '{":id":{"S":"123"}}'
  ```
**Root Cause**: The Lambda function’s new query uses an invalid partition key, causing a `KeySchema` error in DynamoDB, which results in a 500 error returned to API Gateway.
**Solution**:
- Fix the Lambda function code (e.g., Python):
  ```python
  import boto3
  import json

  def lambda_handler(event, context):
      dynamodb = boto3.resource('dynamodb')
      table = dynamodb.Table('Users')
      try:
          user_id = event['queryStringParameters']['userId']
          response = table.query(
              KeyConditionExpression='userId = :id',
              ExpressionAttributeValues={':id': user_id}
          )
          return {
              'statusCode': 200,
              'body': json.dumps(response['Items'])
          }
      except Exception as e:
          return {
              'statusCode': 500,
              'body': json.dumps({'error': str(e)})
          }
  ```
- Ensure the DynamoDB table has the correct partition key (`userId`).
- Verify IAM role permissions:
  ```json
  {
      "Effect": "Allow",
      "Action": ["dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:us-east-1:<account-id>:table/Users"
  }
  ```
- Increase Lambda timeout if queries are slow:
  ```bash
  aws lambda update-function-configuration --function-name my-lambda --timeout 15
  ```
- Monitor CloudWatch for `Errors` and DynamoDB `ThrottledRequests`:
  ```bash
  aws cloudwatch put-metric-alarm --alarm-name LambdaErrors \
      --metric-name Errors --namespace AWS/Lambda --threshold 1
  ```
- Test the updated function via API Gateway.
**Interview Tips**:
- Explain Lambda error handling and DynamoDB key schemas.
- Discuss IAM least privilege and CloudWatch monitoring.
- Highlight DynamoDB throttling mitigation (e.g., auto-scaling).

### Problem 6: 504 Gateway Timeout in API Gateway with Step Functions (Server-Side) [New]
**Context**: A workflow application uses API Gateway to trigger an AWS Step Functions state machine for order processing. During peak hours, some requests return 504 Gateway Timeout errors, delaying order confirmations.
**Challenge**: Resolve the 504 errors to ensure timely order processing.
**Troubleshooting**:
1. Check CloudWatch Logs for API Gateway to confirm timeout errors.
2. Review Step Functions execution logs for duration and errors.
3. Inspect the Lambda functions within the state machine for performance bottlenecks.
4. Verify API Gateway integration timeout settings (default: 29 seconds).
5. Test the state machine execution manually:
  ```bash
  aws stepfunctions start-execution --state-machine-arn <arn> --input '{"orderId": "123"}'
  ```
**Root Cause**: The Step Functions state machine includes a Lambda function with a complex database query that exceeds API Gateway’s 29-second timeout during peak load.
**Solution**:
- Optimize the Lambda function (e.g., Python):
  ```python
  import boto3
  def lambda_handler(event, context):
      dynamodb = boto3.client('dynamodb')
      order_id = event['orderId']
      # Optimize query with index
      response = dynamodb.query(
          TableName='Orders',
          IndexName='OrderStatusIndex',
          KeyConditionExpression='orderId = :id',
          ExpressionAttributeValues={':id': {'S': order_id}}
      )
      return {'status': response['Items'][0]['status']['S']}
  ```
- Add a global secondary index to DynamoDB for faster queries:
  ```bash
  aws dynamodb update-table --table-name Orders \
      --global-secondary-index-updates '[{
          "Create": {
              "IndexName": "OrderStatusIndex",
              "KeySchema": [{"AttributeName": "orderId", "KeyType": "HASH"}],
              "Projection": {"ProjectionType": "ALL"}
          }
      }]'
  ```
- Increase API Gateway timeout (max 29 seconds):
  ```bash
  aws apigateway update-integration --rest-api-id <api-id> --resource-id <resource-id> \
      --http-method POST --patch-operations op=replace,path=/timeoutMilliseconds,value=29000
  ```
- For long-running workflows, use asynchronous Step Functions execution:
  ```json
  {
      "StartAt": "ProcessOrder",
      "States": {
          "ProcessOrder": {
              "Type": "Task",
              "Resource": "<lambda-arn>",
              "Next": "Notify",
              "ResultPath": "$.result"
          },
          "Notify": {
              "Type": "Task",
              "Resource": "arn:aws:states:::sns:publish",
              "End": true
          }
      }
  }
  ```
- Return an immediate response from API Gateway, polling Step Functions status:
  ```python
  def lambda_handler(event, context):
      sfn = boto3.client('stepfunctions')
      response = sfn.start_execution(
          stateMachineArn='<arn>',
          input=json.dumps(event['body'])
      )
      return {'statusCode': 202, 'body': json.dumps({'executionArn': response['executionArn']})}
  ```
- Monitor CloudWatch for `ExecutionTime` in Step Functions and `5xxErrorRate` in API Gateway.
**Interview Tips**:
- Explain API Gateway’s 29-second timeout limitation.
- Discuss synchronous vs asynchronous Step Functions.
- Highlight database optimization and monitoring.

### Problem 7: 401 Unauthorized in Cognito-Protected API Gateway (Client-Side) [New]
**Context**: A web application uses API Gateway with Amazon Cognito for user authentication. After integrating a new frontend, some users receive 401 Unauthorized errors when accessing protected endpoints, despite valid login credentials.
**Challenge**: Fix the 401 errors to restore access for all users.
**Troubleshooting**:
1. Check CloudWatch Logs for API Gateway to identify authentication errors.
2. Verify Cognito User Pool configuration in the AWS Console (e.g., client ID, scopes).
3. Inspect frontend code for correct token handling (e.g., Authorization header).
4. Test authentication with a valid JWT token using Postman.
5. Review IAM roles and Cognito authorizer settings in API Gateway.
**Root Cause**: The frontend incorrectly sends an ID token instead of an access token in the Authorization header, which API Gateway’s Cognito authorizer rejects.
**Solution**:
- Update frontend code to use the access token (e.g., JavaScript with AWS Amplify):
  ```javascript
  import { Auth } from 'aws-amplify';
  async function callApi() {
      const session = await Auth.currentSession();
      const accessToken = session.getAccessToken().getJwtToken();
      const response = await fetch('https://<api-id>.execute-api.us-east-1.amazonaws.com/prod', {
          headers: { Authorization: `Bearer ${accessToken}` }
      });
      return response.json();
  }
  ```
- Ensure API Gateway uses a Cognito authorizer:
  ```bash
  aws apigateway create-authorizer --rest-api-id <api-id> --name cognito-auth \
      --type COGNITO_USER_POOLS --provider-arns <cognito-user-pool-arn> \
      --identity-source 'method.request.header.Authorization'
  ```
- Verify Cognito User Pool client settings (e.g., enabled scopes):
  ```bash
  aws cognito-idp update-user-pool-client --user-pool-id <pool-id> --client-id <client-id> \
      --allowed-oauth-scopes "openid profile"
  ```
- Log authentication failures in CloudWatch:
  ```bash
  aws apigateway update-stage --rest-api-id <api-id> --stage-name prod \
      --patch-operations op=replace,path=/accessLogSettings/destinationArn,value=<log-group-arn>
  ```
- Test with a valid access token to confirm resolution.
- Set CloudWatch alarm for `4xxErrorRate > 3%`.
**Interview Tips**:
- Explain ID token vs access token in Cognito.
- Discuss Cognito authorizer setup and debugging.
- Highlight secure token handling in frontends.

---

## 4. AWS Storage, Networking, and Dockerfiles: Real-Time Interview Problems

### Problem 1: Shared Storage for Microservices
**Context**: CMS on EC2 Auto Scaling Group needs shared media storage, but EBS causes data inconsistency.
**Challenge**: Ensure consistent file access.
**Troubleshooting**:
1. Verify EBS setup.
2. Check Auto Scaling lifecycle hooks.
3. Review app logs for file errors.
**Root Cause**: EBS is single-instance.
**Solution**:
- Use EFS:
  ```bash
  aws efs create-file-system --region us-east-1 --performance-mode generalPurpose
  sudo yum install -y amazon-efs-utils
  sudo mount -t efs <fs-id>:/ /mnt/efs
  ```
- Configure EFS mount targets in multi-AZs.
- Update Auto Scaling launch template.
- Allow NFS (port 2049) in security group.
- Monitor EFS `BurstCreditBalance`.
**Interview Tips**: Compare EBS/EFS/S3, discuss EFS performance.

### Problem 2: Networking Issue Causing Downtime
**Context**: Web app on EC2 in private subnet with ALB returns 503 errors post-network change.
**Challenge**: Restore availability.
**Troubleshooting**:
1. Check ALB access logs for 503s.
2. Verify private subnet route tables.
3. Inspect security groups, VPC Flow Logs.
**Root Cause**: Missing NAT Gateway route.
**Solution**:
- Add NAT Gateway route:
  ```bash
  aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <nat-id>
  ```
- Verify security groups (ALB: HTTP:80, EC2: allow ALB).
- Enable Flow Logs.
- Monitor `HTTPCode_ELB_5XX`.
**Interview Tips**: Explain VPC routing, security groups vs NACLs.

### Problem 3: Optimizing Dockerfile for Legacy Web Server
**Context**: Legacy PHP app on Apache needs containerization for ECS, but Dockerfile is bloated and insecure.
**Challenge**: Optimize for security, size, IPv6.
**Troubleshooting**:
1. Review Dockerfile for inefficiencies.
2. Check security scans for root user.
3. Test IPv6 support.
**Root Cause**: Large image, root user, no IPv6.
**Solution**:
- Optimized Dockerfile:
  ```dockerfile
  FROM php:7.4-apache AS builder
  WORKDIR /var/www/html
  COPY composer.json .
  RUN apt-get update && apt-get install -y unzip \
      && composer install --no-dev --optimize-autoloader \
      && rm -rf /var/lib/apt/lists/*

  FROM php:7.4-apache
  WORKDIR /var/www/html
  RUN apt-get update && apt-get install -y curl \
      && rm -rf /var/lib/apt/lists/* \
      && useradd -m appuser \
      && chown -R appuser:appuser /var/www/html \
      && a2enmod rewrite
  COPY --from=builder /var/www/html/vendor /var/www/html/vendor
  COPY . .
  ENV APACHE_RUN_USER=appuser APACHE_RUN_GROUP=appuser
  RUN sed -i 's/Listen 80/Listen [::]:80/' /etc/apache2/ports.conf
  HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost/health.php || exit 1
  EXPOSE 80
  USER appuser
  CMD ["apache2-foreground"]
  ```
- Deploy to ECS with IPv6 task definition.
- Push to ECR, scan with Inspector.
**Interview Tips**: Discuss multi-stage builds, non-root users, IPv6.

---

## 5. Complex Web Dockerfile with Gunicorn, WSGI, and IPv6: Expanded Examples

**Context**: Production-grade Python web applications using WSGI/ASGI servers (Gunicorn, Uvicorn) with IPv6 support (`[::]:80`). Examples include Flask with Gunicorn, Django with Gunicorn, FastAPI with Uvicorn, and a multi-container setup with Nginx.

### Example 1: Flask with Gunicorn
**Dockerfile**:
```dockerfile
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m appuser \
    && chown -R appuser:appuser /app
COPY --from=builder /root/.local /home/appuser/.local
COPY . .
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH=/home/appuser/.local/bin:$PATH \
    GUNICORN_WORKERS=4 GUNICORN_THREADS=2 GUNICORN_TIMEOUT=120 GUNICORN_LOGLEVEL=info \
    GUNICORN_BIND="[::]:80"
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1
EXPOSE 80
CMD ["gunicorn", "--workers", "$GUNICORN_WORKERS", "--threads", "$GUNICORN_THREADS", \
     "--timeout", "$GUNICORN_TIMEOUT", "--log-level", "$GUNICORN_LOGLEVEL", \
     "--access-logfile", "-", "--error-logfile", "-", "--bind", "$GUNICORN_BIND", "app:app"]
```
**requirements.txt**:
```
flask==2.0.1
gunicorn==20.1.0
```
**app.py**:
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return {'message': 'Welcome to the API'}, 200

@app.route('/health')
def health():
    return {'status': 'healthy'}, 200
```
**AWS Integration**: ECS Fargate, ALB, CloudWatch, CloudFront, Secrets Manager.

### Example 2: Django with Gunicorn
**Dockerfile**:
```dockerfile
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m appuser \
    && chown -R appuser:appuser /app
COPY --from=builder /root/.local /home/appuser/.local
COPY . .
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH=/home/appuser/.local/bin:$PATH \
    GUNICORN_WORKERS=4 GUNICORN_THREADS=2 GUNICORN_TIMEOUT=120 GUNICORN_LOGLEVEL=info \
    GUNICORN_BIND="[::]:80" DJANGO_SETTINGS_MODULE=myproject.settings
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1
EXPOSE 80
CMD ["gunicorn", "--workers", "$GUNICORN_WORKERS", "--threads", "$GUNICORN_THREADS", \
     "--timeout", "$GUNICORN_TIMEOUT", "--log-level", "$GUNICORN_LOGLEVEL", \
     "--access-logfile", "-", "--error-logfile", "-", "--bind", "$GUNICORN_BIND", \
     "myproject.wsgi:application"]
```
**requirements.txt**:
```
django==3.2.9
gunicorn==20.1.0
psycopg2-binary==2.9.2
```
**AWS Integration**: RDS, S3, CloudFront, ECS Fargate, CloudWatch.

### Example 3: FastAPI with Uvicorn
**Dockerfile**:
```dockerfile
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m appuser \
    && chown -R appuser:appuser /app
COPY --from=builder /root/.local /home/appuser/.local
COPY . .
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH=/home/appuser/.local/bin:$PATH \
    UVICORN_WORKERS=4 UVICORN_HOST="[::]:80" UVICORN_PORT=80
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1
EXPOSE 80
CMD ["uvicorn", "main:app", "--workers", "$UVICORN_WORKERS", "--host", "$UVICORN_HOST", \
     "--port", "$UVICORN_PORT", "--log-level", "info"]
```
**requirements.txt**:
```
fastapi==0.68.1
uvicorn==0.15.0
```
**main.py**:
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
async def home():
    return {"message": "Welcome to the API"}

@app.get("/health")
async def health():
    return {"status": "healthy"}
```
**AWS Integration**: API Gateway, DynamoDB, ECS Fargate, CloudWatch, CloudFront.

### Example 4: Multi-Container with Nginx
**Dockerfile (Flask)**:
```dockerfile
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m appuser \
    && chown -R appuser:appuser /app
COPY --from=builder /root/.local /home/appuser/.local
COPY . .
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH=/home/appuser/.local/bin:$PATH \
    GUNICORN_WORKERS=4 GUNICORN_THREADS=2 GUNICORN_TIMEOUT=120 GUNICORN_LOGLEVEL=info \
    GUNICORN_BIND="0.0.0.0:8000"
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
EXPOSE 8000
CMD ["gunicorn", "--workers", "$GUNICORN_WORKERS", "--threads", "$GUNICORN_THREADS", \
     "--timeout", "$GUNICORN_TIMEOUT", "--log-level", "$GUNICORN_LOGLEVEL", \
     "--access-logfile", "-", "--error-logfile", "-", "--bind", "$GUNICORN_BIND", "app:app"]
```
**Dockerfile (Nginx)**:
```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
RUN adduser -D appuser && chown -R appuser:appuser /etc/nginx
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --ret