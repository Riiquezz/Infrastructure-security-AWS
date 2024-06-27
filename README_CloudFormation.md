# CloudFormation Example

# 1. Pick an infrastructure as a code tool of choice
CloudFormation

# 2. Provision example single page application following the architecture diagram below
AWS CloudFormation Template
```
AWSTemplateFormatVersion: '2010-09-09'
Description: Provisioning SPA with CloudFront, S3, API Gateway, and Lambda

Resources:
  SPAHostingBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: PublicRead

  SPAHostingBucketPolicy:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Ref SPAHostingBucket
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal: '*'
            Action: 's3:GetObject'
            Resource: !Sub '${SPAHostingBucket.Arn}/*'

  CloudFrontDistribution:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Origins:
          - Id: S3Origin
            DomainName: !GetAtt SPAHostingBucket.DomainName
            S3OriginConfig:
              OriginAccessIdentity: ''
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods:
            - GET
            - HEAD
          CachedMethods:
            - GET
            - HEAD
        Enabled: true

  LambdaExecutionRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: LambdaExecutionPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'

  LambdaFunction:
    Type: 'AWS::Lambda::Function'
    Properties:
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              return {
                  'statusCode': 200,
                  'body': 'Hello from Lambda!'
              }
      Runtime: python3.8

  ApiGateway:
    Type: 'AWS::ApiGateway::RestApi'
    Properties:
      Name: 'SPA API'

  ApiGatewayResource:
    Type: 'AWS::ApiGateway::Resource'
    Properties:
      ParentId: !GetAtt ApiGateway.RootResourceId
      PathPart: 'lambda'
      RestApiId: !Ref ApiGateway

  ApiGatewayMethod:
    Type: 'AWS::ApiGateway::Method'
    Properties:
      AuthorizationType: 'NONE'
      HttpMethod: 'GET'
      ResourceId: !Ref ApiGatewayResource
      RestApiId: !Ref ApiGateway
      Integration:
        IntegrationHttpMethod: 'POST'
        Type: 'AWS_PROXY'
        Uri: !Sub
          - arn:aws:apigateway:${Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations
          - Region: !Ref 'AWS::Region'

  LambdaInvokePermission:
    Type: 'AWS::Lambda::Permission'
    Properties:
      Action: 'lambda:InvokeFunction'
      FunctionName: !Ref LambdaFunction
      Principal: 'apigateway.amazonaws.com'
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGateway}/*/*/lambda'

```

# 3. Write down security recommendation for each service
## S3 Bucket
- Enable server-side encryption to protect data at rest.
- Use S3 bucket policies and IAM policies to restrict access to the bucket.
- Enable versioning to preserve and recover overwritten and deleted objects.

## CloudFront
- Use HTTPS to ensure data in transit is encrypted.
- Restrict access to S3 bucket using Origin Access Identity (OAI).
- Configure Geo-Restriction to block access from specific countries if required.

## API Gateway
- Enable API Gateway caching to improve performance and reduce the load on backend services.
- Use AWS WAF (Web Application Firewall) to protect against common web exploits.
- Implement usage plans and API keys to control access to the API.

## Lambda
- Use IAM roles with the least privilege to limit the permissions granted to Lambda functions.
- Enable encryption for environment variables.
- Ensure Lambda function code is regularly reviewed and updated.

# 4. Suggest logging and monitoring
## S3 Bucket
- Enable server access logging to track requests made to your bucket.
- Use AWS CloudTrail to log S3 bucket activity for auditing.

## CloudFront
- Enable CloudFront access logs to capture detailed information about every user request.
- Use AWS CloudWatch to monitor CloudFront metrics like cache hit rate and request count.

## API Gateway
- Enable CloudWatch Logs for API Gateway to capture detailed request and response data.
- Set up CloudWatch Alarms to monitor API Gateway metrics like 4XX and 5XX error rates.

## Lambda
- Use CloudWatch Logs to monitor and troubleshoot Lambda function executions.
- Set up CloudWatch Alarms for Lambda function errors and throttling.
- Use AWS X-Ray to trace and analyze requests as they travel through your application.
